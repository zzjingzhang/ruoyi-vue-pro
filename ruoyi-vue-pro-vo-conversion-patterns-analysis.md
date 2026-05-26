# ruoyi-vue-pro VO 转换模式分析

## 1. 概述

ruoyi-vue-pro 是一个基于 Spring Boot 的快速开发框架，采用分层架构设计。在项目中，对象转换是一个核心环节，主要涉及以下几类对象：

- **DO（Data Object）**：数据库实体对象，与数据库表结构一一对应
- **CreateReqVO**：创建请求 VO，用于接收前端创建数据的请求参数
- **UpdateReqVO**：更新请求 VO，用于接收前端更新数据的请求参数
- **RespVO**：响应 VO，用于返回给前端的数据
- **ExcelVO**：Excel 导入导出 VO，用于 Excel 文件的读取和写入
- **AppVO**：移动端应用 VO，专门用于移动端接口

项目主要使用两种对象转换方式：
1. **MapStruct**：编译期生成转换代码，性能高，类型安全
2. **BeanUtils**：运行时反射转换，使用简单灵活

## 2. MapStruct 或 Convert 接口定义位置

在 ruoyi-vue-pro 中，MapStruct 的 Convert 接口定义在各个模块的 `convert` 包下，遵循统一的命名规范和结构。

### 2.1 位置规范

- **包路径**：`cn.iocoder.yudao.module.{模块名}.convert.{业务名}.{业务名}Convert.java`
- **文件命名**：`{业务名}Convert.java`
- **接口定义**：使用 `@Mapper` 注解标记

### 2.2 典型定义示例

以 `MemberUserConvert.java` 为例：

```java
package cn.iocoder.yudao.module.member.convert.user;

import cn.iocoder.yudao.framework.common.pojo.PageResult;
import cn.iocoder.yudao.module.member.api.user.dto.MemberUserRespDTO;
import cn.iocoder.yudao.module.member.controller.admin.user.vo.MemberUserRespVO;
import cn.iocoder.yudao.module.member.controller.admin.user.vo.MemberUserUpdateReqVO;
import cn.iocoder.yudao.module.member.controller.app.user.vo.AppMemberUserInfoRespVO;
import cn.iocoder.yudao.module.member.convert.address.AddressConvert;
import cn.iocoder.yudao.module.member.dal.dataobject.group.MemberGroupDO;
import cn.iocoder.yudao.module.member.dal.dataobject.level.MemberLevelDO;
import cn.iocoder.yudao.module.member.dal.dataobject.tag.MemberTagDO;
import cn.iocoder.yudao.module.member.dal.dataobject.user.MemberUserDO;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.Mappings;
import org.mapstruct.factory.Mappers;

import java.util.List;
import java.util.Map;

import static cn.iocoder.yudao.framework.common.util.collection.CollectionUtils.convertList;
import static cn.iocoder.yudao.framework.common.util.collection.CollectionUtils.convertMap;

@Mapper(uses = {AddressConvert.class})
public interface MemberUserConvert {

    MemberUserConvert INSTANCE = Mappers.getMapper(MemberUserConvert.class);

    AppMemberUserInfoRespVO convert(MemberUserDO bean);

    @Mappings({
            @Mapping(source = "level", target = "level"),
            @Mapping(source = "bean.id", target = "id"),
            @Mapping(source = "bean.experience", target = "experience")
    })
    AppMemberUserInfoRespVO convert(MemberUserDO bean, MemberLevelDO level);

    MemberUserRespDTO convert2(MemberUserDO bean);

    List<MemberUserRespDTO> convertList2(List<MemberUserDO> list);

    MemberUserDO convert(MemberUserUpdateReqVO bean);

    PageResult<MemberUserRespVO> convertPage(PageResult<MemberUserDO> page);

    @Mapping(source = "areaId", target = "areaName", qualifiedByName = "convertAreaIdToAreaName")
    MemberUserRespVO convert03(MemberUserDO bean);

    default PageResult<MemberUserRespVO> convertPage(PageResult<MemberUserDO> pageResult,
                                                     List<MemberTagDO> tags,
                                                     List<MemberLevelDO> levels,
                                                     List<MemberGroupDO> groups) {
        PageResult<MemberUserRespVO> result = convertPage(pageResult);
        // 处理关联数据
        Map<Long, String> tagMap = convertMap(tags, MemberTagDO::getId, MemberTagDO::getName);
        Map<Long, String> levelMap = convertMap(levels, MemberLevelDO::getId, MemberLevelDO::getName);
        Map<Long, String> groupMap = convertMap(groups, MemberGroupDO::getId, MemberGroupDO::getName);
        // 填充关联数据
        result.getList().forEach(user -> {
            user.setTagNames(convertList(user.getTagIds(), tagMap::get));
            user.setLevelName(levelMap.get(user.getLevelId()));
            user.setGroupName(groupMap.get(user.getGroupId()));
        });
        return result;
    }
}
```

### 2.3 关键特征

1. **单例模式**：使用 `Mappers.getMapper(xxx.class)` 获取实例
2. **default 方法**：支持复杂的自定义转换逻辑
3. **@Mapper 注解**：标记为 MapStruct 接口，支持 `uses` 引入其他转换器
4. **@Mapping 注解**：显式指定字段映射关系
5. **方法命名规范**：
   - `convert()`：基本转换
   - `convertList()`：列表转换
   - `convertPage()`：分页结果转换
   - `convertExcel()`：Excel 转换

## 3. 哪些字段不会自动转换

MapStruct 虽然功能强大，但并非所有字段都能自动转换。以下是需要注意的场景：

### 3.1 字段名不匹配

当源对象和目标对象的字段名不一致时，MapStruct 无法自动映射，需要使用 `@Mapping` 注解显式指定：

```java
@Mapping(source = "userName", target = "name")
UserRespVO convert(UserDO user);
```

### 3.2 类型不兼容

当源字段类型与目标字段类型不兼容时，需要自定义转换器：

**示例 1：基本类型转换**
```java
// UserDO.status 是 Integer 类型
// UserRespVO.statusName 是 String 类型
// 这种情况无法自动转换，需要自定义逻辑
```

**示例 2：复杂对象转换**
```java
// UserDO.deptId 是 Long 类型
// UserRespVO.deptName 是 String 类型
// 这种情况需要查询数据库获取部门名称
```

### 3.3 关联数据填充

当目标对象需要包含关联对象的信息时，需要在 `default` 方法中手动处理：

```java
default UserRespVO convert(UserDO user, DeptDO dept) {
    UserRespVO userVO = BeanUtils.toBean(user, UserRespVO.class);
    if (dept != null) {
        userVO.setDeptName(dept.getName()); // 手动填充关联数据
    }
    return userVO;
}
```

### 3.4 特殊处理字段

- **密码字段**：DO 中存储的是加密密码，RespVO 中不应该返回
- **敏感信息**：身份证号、手机号等可能需要脱敏
- **审计字段**：createdAt、updatedAt、createdBy、updatedBy 等可能根据场景决定是否返回
- **内部字段**：如 tenantId、deleted 等不需要暴露给前端

## 4. 字典/枚举/金额/时间字段处理

### 4.1 字典字段处理

项目中使用 `@DictFormat` 注解配合 `DictConvert` 转换器处理字典字段。

**定义位置**：`yudao-framework/yudao-spring-boot-starter-excel/src/main/java/cn/iocoder/yudao/framework/excel/core/convert/DictConvert.java`

```java
@Slf4j
public class DictConvert implements Converter<Object> {

    @Override
    public WriteCellData<String> convertToExcelData(Object object, ExcelContentProperty contentProperty,
                                                    GlobalConfiguration globalConfiguration) {
        // 空时，返回空
        if (object == null) {
            return new WriteCellData<>("");
        }

        // 使用字典格式化
        String type = getType(contentProperty);
        String value = String.valueOf(object);
        String label = DictFrameworkUtils.parseDictDataLabel(type, value);
        if (label == null) {
            log.error("[convertToExcelData][type({}) 转换不了 label({})]", type, value);
            return new WriteCellData<>("");
        }
        // 生成 Excel 小表格
        return new WriteCellData<>(label);
    }

    private static String getType(ExcelContentProperty contentProperty) {
        return contentProperty.getField().getAnnotation(DictFormat.class).value();
    }
}
```

**使用示例**：
```java
@ExcelProperty(value = "用户状态", converter = DictConvert.class)
@DictFormat(DictTypeConstants.COMMON_STATUS)
private Integer status;
```

### 4.2 枚举字段处理

枚举字段的处理方式：

1. **直接存储枚举值**：将枚举的 `code` 或 `value` 存储到数据库
2. **转换为描述**：在返回给前端时，转换为枚举的描述信息

**DO 示例**：
```java
/**
 * 帐号状态
 *
 * 枚举 {@link CommonStatusEnum}
 */
private Integer status;
```

**RespVO 处理**：
```java
// 方式 1：直接返回状态码
private Integer status;

// 方式 2：返回状态描述（需要在 Convert 中处理）
private String statusName;
```

### 4.3 金额字段处理

项目中金额通常以分为单位存储（Integer 类型），在展示时转换为元。

**定义位置**：`yudao-framework/yudao-spring-boot-starter-excel/src/main/java/cn/iocoder/yudao/framework/excel/core/convert/MoneyConvert.java`

```java
public class MoneyConvert implements Converter<Integer> {

    @Override
    public WriteCellData<String> convertToExcelData(Integer value, ExcelContentProperty contentProperty,
                                                    GlobalConfiguration globalConfiguration) {
        BigDecimal result = BigDecimal.valueOf(value)
                .divide(new BigDecimal(100), 2, RoundingMode.HALF_UP);
        return new WriteCellData<>(result.toString());
    }
}
```

**使用示例**：
```java
@ExcelProperty(value = "支付金额", converter = MoneyConvert.class)
private Integer price;
```

**转换逻辑**：
- **存储**：100 元 → 存储为 10000（分）
- **展示**：10000 → 转换为 "100.00"（元）

### 4.4 时间字段处理

项目中使用 `LocalDateTime` 作为时间类型，通过注解控制格式化。

**DO 示例**：
```java
/**
 * 最后登录时间
 */
private LocalDateTime loginDate;

/**
 * 出生日期
 */
private LocalDateTime birthday;
```

**ReqVO 注解**：
```java
import org.springframework.format.annotation.DateTimeFormat;

@DateTimeFormat(pattern = FORMAT_YEAR_MONTH_DAY)
private LocalDateTime birthday;
```

**RespVO 注解**：
```java
import com.fasterxml.jackson.annotation.JsonFormat;

@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "GMT+8")
private LocalDateTime createTime;
```

## 5. 列表转换和分页转换的差异

### 5.1 列表转换（List Conversion）

列表转换用于将 `List<DO>` 转换为 `List<VO>`。

**Convert 接口定义**：
```java
List<PayOrderExcelVO> convertList(List<PayOrderDO> list, Map<Long, PayAppDO> appMap);
```

**实现方式**：
```java
default List<PayOrderExcelVO> convertList(List<PayOrderDO> list, Map<Long, PayAppDO> appMap) {
    return CollectionUtils.convertList(list, order -> {
        PayOrderExcelVO excelVO = convertExcel(order);
        MapUtils.findAndThen(appMap, order.getAppId(), app -> excelVO.setAppName(app.getName()));
        return excelVO;
    });
}
```

**使用场景**：
- 导出 Excel 数据
- 批量查询返回
- 下拉列表数据

### 5.2 分页转换（Page Conversion）

分页转换用于将 `PageResult<DO>` 转换为 `PageResult<VO>`，保持分页信息。

**PageResult 结构**：
```java
public class PageResult<T> {
    private List<T> list;      // 数据列表
    private Long total;        // 总记录数
}
```

**Convert 接口定义**：
```java
PageResult<MemberUserRespVO> convertPage(PageResult<MemberUserDO> page);

default PageResult<MemberUserRespVO> convertPage(PageResult<MemberUserDO> pageResult,
                                                 List<MemberTagDO> tags,
                                                 List<MemberLevelDO> levels,
                                                 List<MemberGroupDO> groups) {
    // 1. 先进行基础转换
    PageResult<MemberUserRespVO> result = convertPage(pageResult);
    
    // 2. 处理关联数据
    Map<Long, String> tagMap = convertMap(tags, MemberTagDO::getId, MemberTagDO::getName);
    Map<Long, String> levelMap = convertMap(levels, MemberLevelDO::getId, MemberLevelDO::getName);
    Map<Long, String> groupMap = convertMap(groups, MemberGroupDO::getId, MemberGroupDO::getName);
    
    // 3. 填充关联数据
    result.getList().forEach(user -> {
        user.setTagNames(convertList(user.getTagIds(), tagMap::get));
        user.setLevelName(levelMap.get(user.getLevelId()));
        user.setGroupName(groupMap.get(user.getGroupId()));
    });
    
    return result;
}
```

**BeanUtils 实现**：
```java
public static <S, T> PageResult<T> toBean(PageResult<S> source, Class<T> targetType) {
    if (source == null) {
        return null;
    }
    List<T> list = toBean(source.getList(), targetType);
    return new PageResult<>(list, source.getTotal());
}
```

### 5.3 核心差异

| 特性 | 列表转换 | 分页转换 |
|------|---------|---------|
| 输入类型 | `List<DO>` | `PageResult<DO>` |
| 输出类型 | `List<VO>` | `PageResult<VO>` |
| 包含信息 | 仅数据列表 | 数据列表 + 总记录数 |
| 典型场景 | Excel 导出、批量查询 | 分页查询接口 |
| 关注点 | 数据转换本身 | 保持分页信息完整性 |

## 6. UpdateReqVO 直接覆盖 DO 可能导致字段被置空的原因

### 6.1 问题描述

在更新操作中，如果直接使用 `BeanUtils.copyProperties()` 或 MapStruct 转换将 UpdateReqVO 的值覆盖到 DO 上，可能导致原本有值的字段被置为 `null`。

### 6.2 根本原因

#### 原因 1：UpdateReqVO 字段不全

UpdateReqVO 通常只包含需要更新的字段，而 DO 包含所有字段。

**示例**：
```java
// MemberUserDO（部分字段）
public class MemberUserDO extends TenantBaseDO {
    private Long id;
    private String mobile;
    private String password;      // 加密后的密码
    private Integer status;
    private String registerIp;    // 注册 IP
    private LocalDateTime loginDate; // 最后登录时间
    private String nickname;
    private Integer point;        // 积分
    private Integer experience;   // 经验值
    // ... 其他字段
}

// MemberUserUpdateReqVO
public class MemberUserUpdateReqVO extends MemberUserBaseVO {
    private Long id;
}

// MemberUserBaseVO
public class MemberUserBaseVO {
    private String mobile;
    private Byte status;
    private String nickname;
    private String avatar;
    private String name;
    private Integer sex;
    private Long areaId;
    private LocalDateTime birthday;
    private String mark;
    private List<Long> tagIds;
    private Long levelId;
    private Long groupId;
}
```

**问题**：UpdateReqVO 不包含 `password`、`registerIp`、`loginDate`、`point`、`experience` 等字段。

#### 原因 2：BeanUtils.copyProperties 会复制 null 值

项目中的 `BeanUtils.copyProperties` 实现：

```java
public static void copyProperties(Object source, Object target) {
    if (source == null || target == null) {
        return;
    }
    BeanUtil.copyProperties(source, target, false);
}
```

**关键参数**：第三个参数 `false` 表示 `ignoreNullValue = false`，即会复制 null 值。

#### 原因 3：MapStruct 也会设置 null 值

MapStruct 生成的转换代码会为所有字段赋值，包括 null 值。

### 6.3 实际示例：错误的更新方式

**错误代码**：
```java
@Override
@Transactional(rollbackFor = Exception.class)
public void updateUser(MemberUserUpdateReqVO updateReqVO) {
    // 校验存在
    validateUserExists(updateReqVO.getId());
    // 校验手机唯一
    validateMobileUnique(updateReqVO.getId(), updateReqVO.getMobile());

    // 更新 - 错误方式：直接转换并更新
    MemberUserDO updateObj = MemberUserConvert.INSTANCE.convert(updateReqVO);
    memberUserMapper.updateById(updateObj);
}
```

**实际生成的 SQL**（MyBatis-Plus 的 `updateById`）：
```sql
UPDATE member_user 
SET mobile = '13800138000',
    status = 1,
    nickname = '张三',
    avatar = 'https://xxx.com/avatar.png',
    name = '张三',
    sex = 1,
    area_id = NULL,        -- 被置空了！
    birthday = NULL,       -- 被置空了！
    mark = NULL,           -- 被置空了！
    tag_ids = NULL,        -- 被置空了！
    level_id = NULL,       -- 被置空了！
    group_id = NULL,       -- 被置空了！
    update_time = '2024-01-01 12:00:00',
    updater = 1
WHERE id = 100;
```

**后果**：用户原本设置的 `areaId`、`birthday`、`mark`、`tagIds`、`levelId`、`groupId` 等字段都被置空了。

## 7. 正确的 merge 策略

### 7.1 方案一：先查询再更新（推荐）

**核心思想**：
1. 先从数据库查询出完整的 DO
2. 只复制 UpdateReqVO 中不为 null 的字段
3. 保存更新后的 DO

**正确代码**：
```java
@Override
@Transactional(rollbackFor = Exception.class)
public void updateUser(MemberUserUpdateReqVO updateReqVO) {
    // 1. 校验存在
    MemberUserDO user = validateUserExists(updateReqVO.getId());
    // 2. 校验手机唯一
    validateMobileUnique(updateReqVO.getId(), updateReqVO.getMobile());

    // 3. 正确方式：先查询，再复制非 null 字段
    // 方式 A：使用 BeanUtils 复制（需要忽略 null 值）
    BeanUtils.copyPropertiesIgnoreNull(updateReqVO, user);
    
    // 方式 B：手动设置需要更新的字段
    // user.setMobile(updateReqVO.getMobile());
    // user.setStatus(updateReqVO.getStatus());
    // ... 其他字段
    
    // 4. 更新
    memberUserMapper.updateById(user);
}
```

### 7.2 方案二：使用 MyBatis-Plus 的 UpdateWrapper

**核心思想**：只更新需要更新的字段。

```java
@Override
@Transactional(rollbackFor = Exception.class)
public void updateUser(MemberUserUpdateReqVO updateReqVO) {
    // 1. 校验存在
    validateUserExists(updateReqVO.getId());
    // 2. 校验手机唯一
    validateMobileUnique(updateReqVO.getId(), updateReqVO.getMobile());

    // 3. 构建 UpdateWrapper，只设置需要更新的字段
    LambdaUpdateWrapper<MemberUserDO> updateWrapper = Wrappers.<MemberUserDO>lambdaUpdate()
            .eq(MemberUserDO::getId, updateReqVO.getId());
    
    // 4. 按需设置字段
    if (updateReqVO.getMobile() != null) {
        updateWrapper.set(MemberUserDO::getMobile, updateReqVO.getMobile());
    }
    if (updateReqVO.getStatus() != null) {
        updateWrapper.set(MemberUserDO::getStatus, updateReqVO.getStatus());
    }
    if (updateReqVO.getNickname() != null) {
        updateWrapper.set(MemberUserDO::getNickname, updateReqVO.getNickname());
    }
    // ... 其他字段

    // 5. 执行更新
    memberUserMapper.update(null, updateWrapper);
}
```

### 7.3 方案三：MapStruct + 自定义 ignoreNull 策略

**定义 MapStruct 转换器**：
```java
@Mapper(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
public interface MemberUserConvert {

    MemberUserConvert INSTANCE = Mappers.getMapper(MemberUserConvert.class);

    // 注意：target 必须是已存在的对象
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "createTime", ignore = true)
    @Mapping(target = "updateTime", ignore = true)
    @Mapping(target = "creator", ignore = true)
    @Mapping(target = "updater", ignore = true)
    @Mapping(target = "deleted", ignore = true)
    @Mapping(target = "tenantId", ignore = true)
    void updateMemberUserDO(MemberUserUpdateReqVO source, @MappingTarget MemberUserDO target);
}
```

**使用方式**：
```java
@Override
@Transactional(rollbackFor = Exception.class)
public void updateUser(MemberUserUpdateReqVO updateReqVO) {
    // 1. 查询完整的 DO
    MemberUserDO user = validateUserExists(updateReqVO.getId());
    // 2. 校验手机唯一
    validateMobileUnique(updateReqVO.getId(), updateReqVO.getMobile());

    // 3. 使用 MapStruct 更新，忽略 null 值
    MemberUserConvert.INSTANCE.updateMemberUserDO(updateReqVO, user);

    // 4. 保存
    memberUserMapper.updateById(user);
}
```

## 8. BeanUtils.copyProperties 的风险

### 8.1 风险一：Null 值覆盖

**问题描述**：`BeanUtils.copyProperties` 默认会复制 null 值，导致目标对象的字段被置空。

**示例**：
```java
// 假设数据库中的 user 有完整信息
MemberUserDO dbUser = memberUserMapper.selectById(100);
// dbUser.getAreaId() = 4371

// 前端只更新了昵称
MemberUserUpdateReqVO reqVO = new MemberUserUpdateReqVO();
reqVO.setId(100L);
reqVO.setNickname("新昵称");
// reqVO.getAreaId() = null（未设置）

// 错误：直接复制
BeanUtils.copyProperties(reqVO, dbUser);

// 结果：dbUser.getAreaId() 变成了 null！
System.out.println(dbUser.getAreaId()); // null
```

### 8.2 风险二：类型不匹配

**问题描述**：当源对象和目标对象的字段类型不匹配时，可能导致转换异常或数据丢失。

**示例**：
```java
// UpdateReqVO 中 status 是 Byte 类型
public class MemberUserUpdateReqVO {
    private Byte status;
}

// DO 中 status 是 Integer 类型
public class MemberUserDO {
    private Integer status;
}

// 可能的问题：自动装箱/拆箱导致的异常
```

### 8.3 风险三：字段名拼写错误

**问题描述**：如果 UpdateReqVO 和 DO 的字段名不一致，BeanUtils 会静默跳过该字段。

**示例**：
```java
// UpdateReqVO
private String userMobile;

// DO
private String mobile;

// 结果：userMobile 不会被复制到 mobile，导致更新无效
```

### 8.4 风险四：性能问题

**问题描述**：BeanUtils 基于反射实现，在大数据量场景下性能较差。

### 8.5 最佳实践建议

1. **优先使用 MapStruct**：编译期生成代码，类型安全，性能高
2. **更新操作必须先查询**：获取完整的 DO 后再更新
3. **忽略 null 值**：使用 `ignoreNullValue = true` 或 `NullValuePropertyMappingStrategy.IGNORE`
4. **显式指定字段映射**：对于关键字段，使用 `@Mapping` 显式指定
5. **单元测试覆盖**：确保转换逻辑正确

## 9. 完整更新接口示例分析

### 9.1 当前项目中的实现

**文件位置**：`yudao-module-member/src/main/java/cn/iocoder/yudao/module/member/service/user/MemberUserServiceImpl.java`

**当前实现（第 234-245 行）**：
```java
@Override
@Transactional(rollbackFor = Exception.class)
public void updateUser(MemberUserUpdateReqVO updateReqVO) {
    // 校验存在
    validateUserExists(updateReqVO.getId());
    // 校验手机唯一
    validateMobileUnique(updateReqVO.getId(), updateReqVO.getMobile());

    // 更新
    MemberUserDO updateObj = MemberUserConvert.INSTANCE.convert(updateReqVO);
    memberUserMapper.updateById(updateObj);
}
```

**问题分析**：
- ✅ 做了存在性校验
- ✅ 做了手机号唯一性校验
- ❌ 没有先查询完整的 DO
- ❌ 直接使用 MapStruct 转换 UpdateReqVO 到新的 DO 对象
- ❌ 会导致 UpdateReqVO 中没有的字段被置空

### 9.2 改进后的实现

```java
@Override
@Transactional(rollbackFor = Exception.class)
public void updateUser(MemberUserUpdateReqVO updateReqVO) {
    // 1. 校验存在并获取完整的 DO
    MemberUserDO user = validateUserExists(updateReqVO.getId());
    // 2. 校验手机唯一
    validateMobileUnique(updateReqVO.getId(), updateReqVO.getMobile());

    // 3. 只复制非 null 字段
    // 方式 A：使用 BeanUtils（需要修改 BeanUtils 实现）
    // BeanUtils.copyPropertiesIgnoreNull(updateReqVO, user);
    
    // 方式 B：使用 MapStruct 的 update 方法
    MemberUserConvert.INSTANCE.updateMemberUserFromReqVO(updateReqVO, user);

    // 4. 更新
    memberUserMapper.updateById(user);
}
```

**需要修改的 MemberUserConvert**：
```java
@Mapper(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
public interface MemberUserConvert {

    MemberUserConvert INSTANCE = Mappers.getMapper(MemberUserConvert.class);

    // 新增：从 UpdateReqVO 更新到已存在的 DO
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "createTime", ignore = true)
    @Mapping(target = "updateTime", ignore = true)
    @Mapping(target = "creator", ignore = true)
    @Mapping(target = "updater", ignore = true)
    @Mapping(target = "deleted", ignore = true)
    @Mapping(target = "tenantId", ignore = true)
    @Mapping(target = "password", ignore = true)
    @Mapping(target = "registerIp", ignore = true)
    @Mapping(target = "registerTerminal", ignore = true)
    @Mapping(target = "loginIp", ignore = true)
    @Mapping(target = "loginDate", ignore = true)
    @Mapping(target = "point", ignore = true)
    @Mapping(target = "experience", ignore = true)
    void updateMemberUserFromReqVO(MemberUserUpdateReqVO source, @MappingTarget MemberUserDO target);
}
```

### 9.3 两种方案对比

| 对比项 | 当前实现（有问题） | 改进方案（推荐） |
|-------|-----------------|----------------|
| 数据查询 | 仅校验存在，不获取数据 | 先查询完整的 DO |
| 更新对象 | 创建新的 DO 对象 | 更新已有的 DO 对象 |
| Null 值处理 | 会复制 null 值 | 忽略 null 值 |
| 字段覆盖 | 未传字段被置空 | 未传字段保持原值 |
| 安全性 | 可能丢失数据 | 数据安全 |
| 代码复杂度 | 简单 | 稍复杂但可控 |

## 10. 总结

### 10.1 转换模式总结

ruoyi-vue-pro 采用了灵活的转换策略：

1. **简单转换**：使用 `BeanUtils.toBean()` 或 MapStruct 自动转换
2. **复杂转换**：使用 MapStruct 的 `default` 方法配合自定义逻辑
3. **列表转换**：遍历列表，逐个转换
4. **分页转换**：保持 PageResult 结构，转换内部列表

### 10.2 更新操作最佳实践

1. **必须先查询**：获取完整的 DO 对象
2. **忽略 null 值**：使用 `NullValuePropertyMappingStrategy.IGNORE` 或 `ignoreNullValue = true`
3. **保护敏感字段**：明确忽略 password、audit 等字段
4. **使用 @MappingTarget**：更新已有的对象而不是创建新对象

### 10.3 常见问题避坑

| 问题 | 原因 | 解决方案 |
|-----|------|---------|
| 字段被置空 | 直接转换 UpdateReqVO 到新 DO | 先查询，再更新，忽略 null |
| 类型转换异常 | 字段类型不匹配 | 使用 MapStruct 或显式转换 |
| 关联数据丢失 | VO 不包含关联字段 | 在 Convert 的 default 方法中处理 |
| 性能问题 | 大量使用 BeanUtils | 优先使用 MapStruct |
| 代码可读性差 | 过多的手动 set/get | 合理使用 MapStruct + default 方法 |

### 10.4 推荐的 Convert 接口模板

```java
@Mapper(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
public interface XxxConvert {

    XxxConvert INSTANCE = Mappers.getMapper(XxxConvert.class);

    // ========== 创建转换 ==========
    XxxDO convert(XxxCreateReqVO reqVO);

    // ========== 更新转换（关键：使用 @MappingTarget） ==========
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "createTime", ignore = true)
    @Mapping(target = "updateTime", ignore = true)
    @Mapping(target = "creator", ignore = true)
    @Mapping(target = "updater", ignore = true)
    @Mapping(target = "deleted", ignore = true)
    @Mapping(target = "tenantId", ignore = true)
    void updateXxxFromReqVO(XxxUpdateReqVO source, @MappingTarget XxxDO target);

    // ========== 查询转换 ==========
    XxxRespVO convert(XxxDO xxx);

    // ========== 分页转换 ==========
    PageResult<XxxRespVO> convertPage(PageResult<XxxDO> page);

    // ========== 列表转换 ==========
    List<XxxRespVO> convertList(List<XxxDO> list);

    // ========== Excel 转换 ==========
    XxxExcelVO convertExcel(XxxDO xxx);

    // ========== 复杂转换（使用 default 方法） ==========
    default PageResult<XxxRespVO> convertPageWithRelation(PageResult<XxxDO> page, 
                                                           Map<Long, OtherDO> relationMap) {
        PageResult<XxxRespVO> result = convertPage(page);
        result.getList().forEach(item -> {
            // 填充关联数据
            OtherDO relation = relationMap.get(item.getRelationId());
            if (relation != null) {
                item.setRelationName(relation.getName());
            }
        });
        return result;
    }
}
```

## 11. 相关文件位置

- **Convert 接口示例**：`yudao-module-member/src/main/java/cn/iocoder/yudao/module/member/convert/user/MemberUserConvert.java`
- **BeanUtils 实现**：`yudao-framework/yudao-common/src/main/java/cn/iocoder/yudao/framework/common/util/object/BeanUtils.java`
- **金额转换器**：`yudao-framework/yudao-spring-boot-starter-excel/src/main/java/cn/iocoder/yudao/framework/excel/core/convert/MoneyConvert.java`
- **字典转换器**：`yudao-framework/yudao-spring-boot-starter-excel/src/main/java/cn/iocoder/yudao/framework/excel/core/convert/DictConvert.java`
- **更新接口示例**：`yudao-module-member/src/main/java/cn/iocoder/yudao/module/member/service/user/MemberUserServiceImpl.java`
- **DO 示例**：`yudao-module-member/src/main/java/cn/iocoder/yudao/module/member/dal/dataobject/user/MemberUserDO.java`
- **UpdateReqVO 示例**：`yudao-module-member/src/main/java/cn/iocoder/yudao/module/member/controller/admin/user/vo/MemberUserUpdateReqVO.java`
