# RuoYi-Vue-Pro 字典数据流转机制完整分析

## 1. 概述

本文档深入分析 RuoYi-Vue-Pro (Yudao) 项目中字典数据从后端配置到前端展示的完整链路。涵盖 `dict_type` 与业务字段的绑定方式、后端枚举与系统字典的边界、前端 `useDict` 或字典组件的数据来源、数字字符串类型不一致导致回显失败的原因、字典更新后缓存刷新机制，以及构造一个实际的 `status` 字段类型不一致 bug 并提供完整修复方案。

## 2. 字典数据架构

### 2.1 数据库层

#### 2.1.1 字典类型表 (system_dict_type)

```sql
CREATE TABLE `system_dict_type` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '字典主键',
  `name` varchar(100) NOT NULL DEFAULT '' COMMENT '字典名称',
  `type` varchar(100) NOT NULL DEFAULT '' COMMENT '字典类型',
  `status` tinyint NOT NULL DEFAULT 0 COMMENT '状态（0正常 1停用）',
  `remark` varchar(500) NULL DEFAULT NULL COMMENT '备注',
  `creator` varchar(64) NULL DEFAULT '' COMMENT '创建者',
  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `updater` varchar(64) NULL DEFAULT '' COMMENT '更新者',
  `update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted` bit(1) NOT NULL DEFAULT b'0' COMMENT '是否删除',
  `deleted_time` datetime NULL DEFAULT NULL COMMENT '删除时间',
  PRIMARY KEY (`id`)
) COMMENT = '字典类型表';
```

#### 2.1.2 字典数据表 (system_dict_data)

```sql
CREATE TABLE `system_dict_data` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '字典编码',
  `sort` int NOT NULL DEFAULT 0 COMMENT '字典排序',
  `label` varchar(100) NOT NULL DEFAULT '' COMMENT '字典标签',
  `value` varchar(100) NOT NULL DEFAULT '' COMMENT '字典键值',
  `dict_type` varchar(100) NOT NULL DEFAULT '' COMMENT '字典类型',
  `status` tinyint NOT NULL DEFAULT 0 COMMENT '状态（0正常 1停用）',
  `color_type` varchar(100) NULL DEFAULT '' COMMENT '颜色类型',
  `css_class` varchar(100) NULL DEFAULT '' COMMENT 'css 样式',
  `remark` varchar(500) NULL DEFAULT NULL COMMENT '备注',
  `creator` varchar(64) NULL DEFAULT '' COMMENT '创建者',
  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `updater` varchar(64) NULL DEFAULT '' COMMENT '更新者',
  `update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted` bit(1) NOT NULL DEFAULT b'0' COMMENT '是否删除',
  PRIMARY KEY (`id`)
) COMMENT = '字典数据表';
```

**关键点**：`value` 字段在数据库中是 `varchar(100)` 字符串类型，但业务表中对应字段可能是 `tinyint` 整数类型，这是类型不一致问题的根源。

### 2.2 后端实体层

#### 2.2.1 字典类型实体 (DictTypeDO)

**路径**: `yudao-module-system/src/main/java/cn/iocoder/yudao/module/system/dal/dataobject/dict/DictTypeDO.java`

```java
@TableName("system_dict_type")
@Data
@EqualsAndHashCode(callSuper = true)
public class DictTypeDO extends BaseDO {
    @TableId
    private Long id;
    private String name;           // 字典名称
    private String type;           // 字典类型（关键标识）
    private Integer status;        // 状态
    private String remark;
    private LocalDateTime deletedTime;
}
```

#### 2.2.2 字典数据实体 (DictDataDO)

**路径**: `yudao-module-system/src/main/java/cn/iocoder/yudao/module/system/dal/dataobject/dict/DictDataDO.java`

```java
@TableName("system_dict_data")
@Data
@EqualsAndHashCode(callSuper = true)
public class DictDataDO extends BaseDO {
    @TableId
    private Long id;
    private Integer sort;          // 字典排序
    private String label;          // 字典标签（展示用）
    private String value;          // 字典值（String 类型！重要）
    private String dictType;       // 字典类型（关联 DictTypeDO.type）
    private Integer status;        // 状态
    private String colorType;      // 颜色类型
    private String cssClass;       // css 样式
    private String remark;
}
```

**关键点**：`DictDataDO.value` 是 `String` 类型，与数据库 `varchar` 对应。

#### 2.2.3 字典数据传输对象 (DictDataRespDTO)

**路径**: `yudao-framework/yudao-common/src/main/java/cn/iocoder/yudao/framework/common/biz/system/dict/dto/DictDataRespDTO.java`

```java
@Data
public class DictDataRespDTO {
    private String label;          // 字典标签
    private String value;          // 字典值（String 类型）
    private String dictType;       // 字典类型
    private Integer status;        // 状态
}
```

### 2.3 字典类型常量

**路径**: `yudao-module-system/src/main/java/cn/iocoder/yudao/module/system/enums/DictTypeConstants.java`

```java
public interface DictTypeConstants {
    String USER_TYPE = "user_type";              // 用户类型
    String COMMON_STATUS = "common_status";      // 系统状态
    String USER_SEX = "system_user_sex";         // 用户性别
    String DATA_SCOPE = "system_data_scope";     // 数据范围
    String LOGIN_TYPE = "system_login_type";     // 登录日志的类型
    String LOGIN_RESULT = "system_login_result"; // 登录结果
    // ... 其他模块的 DictTypeConstants
}
```

---

## 3. dict_type 与业务字段的绑定方式

### 3.1 后端绑定：@DictFormat 注解

业务字段通过 `@DictFormat` 注解与字典类型绑定，主要用于 Excel 导入导出时的字典值翻译。

#### 3.1.1 注解定义

**路径**: `yudao-framework/yudao-spring-boot-starter-excel/src/main/java/cn/iocoder/yudao/framework/excel/core/annotations/DictFormat.java`

```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
public @interface DictFormat {
    /**
     * 字典类型
     * 例如：SysDictTypeConstants、InfDictTypeConstants
     */
    String value();
}
```

#### 3.1.2 使用示例

**路径**: `yudao-module-system/src/main/java/cn/iocoder/yudao/module/system/controller/admin/user/vo/user/UserRespVO.java`

```java
@Schema(description = "管理后台 - 用户信息 Response VO")
@Data
@ExcelIgnoreUnannotated
public class UserRespVO {
    @Schema(description = "用户编号")
    @ExcelProperty("用户编号")
    private Long id;

    @Schema(description = "用户账号")
    @ExcelProperty("用户名称")
    private String username;

    @Schema(description = "用户性别，参见 SexEnum 枚举类", example = "1")
    @ExcelProperty(value = "用户性别", converter = DictConvert.class)
    @DictFormat(DictTypeConstants.USER_SEX)
    private Integer sex;

    @Schema(description = "状态，参见 CommonStatusEnum 枚举类", example = "1")
    @ExcelProperty(value = "帐号状态", converter = DictConvert.class)
    @DictFormat(DictTypeConstants.COMMON_STATUS)
    private Integer status;

    // ... 其他字段
}
```

#### 3.1.3 绑定方式总结

| 绑定维度 | 实现方式 | 关键代码 |
|---------|---------|---------|
| **注解绑定** | `@DictFormat` 标记字段 | `@DictFormat(DictTypeConstants.COMMON_STATUS)` |
| **转换器绑定** | `DictConvert` 类处理 | `@ExcelProperty(..., converter = DictConvert.class)` |
| **常量绑定** | `DictTypeConstants` 定义字典类型常量 | `String COMMON_STATUS = "common_status"` |

### 3.2 代码生成器中的绑定

**路径**: `yudao-module-infra/src/main/resources/codegen/java/controller/vo/respVO.vm`

```vm
## 处理 Excel 导出
import cn.idev.excel.annotation.*;
#foreach ($column in $columns)
#if ("$!column.dictType" != "")## 有设置数据字典
import ${DictFormatClassName};
import ${DictConvertClassName};
#break
#end
#end

## 逐个处理字段
#foreach ($column in $columns)
#if (${column.listOperationResult})
## 1. 处理 Swagger 注解
    @Schema(description = "${column.columnComment}"...)
## 2. 处理 Excel 导出
#if ("$!column.dictType" != "")##处理枚举值
    @ExcelProperty(value = "${column.columnComment}", converter = DictConvert.class)
    @DictFormat("${column.dictType}") // TODO 代码优化：建议设置到对应的 DictTypeConstants 枚举类中
#else
    @ExcelProperty("${column.columnComment}")
#end
## 3. 处理字段定义
    private ${column.javaType} ${column.javaField};
#end
#end
```

### 3.3 前端绑定

前端通过字典类型名称从后端获取字典列表，然后绑定到 select/radio/tag 组件。

#### 3.3.1 后端 API

**路径**: `yudao-module-system/src/main/java/cn/iocoder/yudao/module/system/controller/admin/dict/DictDataController.java`

```java
@RestController
@RequestMapping("/system/dict-data")
@Validated
public class DictDataController {

    @GetMapping(value = {"/list-all-simple", "simple-list"})
    @Operation(summary = "获得全部字典数据列表", description = "一般用于管理后台缓存字典数据在本地")
    public CommonResult<List<DictDataSimpleRespVO>> getSimpleDictDataList() {
        List<DictDataDO> list = dictDataService.getDictDataList(
                CommonStatusEnum.ENABLE.getStatus(), null);
        return success(BeanUtils.toBean(list, DictDataSimpleRespVO.class));
    }
}
```

#### 3.3.2 前端使用流程

1. **获取所有字典**: 前端启动时调用 `/system/dict-data/simple-list` 接口
2. **按类型过滤**: 根据 `dict_type` 过滤出对应字典列表
3. **组件绑定**: 将字典列表绑定到 `<el-select>`、`<el-radio-group>` 等组件

---

## 4. 后端枚举和系统字典的边界

### 4.1 后端枚举 (Java Enum)

#### 4.1.1 定义示例

**路径**: `yudao-framework/yudao-common/src/main/java/cn/iocoder/yudao/framework/common/enums/CommonStatusEnum.java`

```java
@Getter
@AllArgsConstructor
public enum CommonStatusEnum implements ArrayValuable<Integer> {
    ENABLE(0, "开启"),
    DISABLE(1, "关闭");

    public static final Integer[] ARRAYS = Arrays.stream(values())
            .map(CommonStatusEnum::getStatus)
            .toArray(Integer[]::new);

    private final Integer status;  // Integer 类型
    private final String name;

    @Override
    public Integer[] array() {
        return ARRAYS;
    }

    public static boolean isEnable(Integer status) {
        return ObjUtil.equal(ENABLE.status, status);
    }

    public static boolean isDisable(Integer status) {
        return ObjUtil.equal(DISABLE.status, status);
    }
}
```

#### 4.1.2 后端枚举的使用场景

| 场景 | 说明 | 示例 |
|-----|------|------|
| **代码内逻辑判断** | 业务逻辑中的条件判断 | `if (CommonStatusEnum.ENABLE.getStatus().equals(user.getStatus()))` |
| **参数校验** | 校验入参是否为有效值 | `@InEnum(CommonStatusEnum.class)` |
| **静态常量** | 编译期确定的固定值 | 开关状态、性别、数据范围 |
| **类型安全** | 编译器检查，避免魔法值 | `CommonStatusEnum.ENABLE` 而非 `0` |

### 4.2 系统字典 (Database Dict)

#### 4.2.1 数据库存储示例

```sql
-- 字典类型
INSERT INTO `system_dict_type` (`id`, `name`, `type`, `status`) 
VALUES (10, '系统状态', 'common_status', 0);

-- 字典数据
INSERT INTO `system_dict_data` (`id`, `sort`, `label`, `value`, `dict_type`, `status`, `color_type`) 
VALUES (27, 1, '开启', '0', 'common_status', 0, 'primary');
INSERT INTO `system_dict_data` (`id`, `sort`, `label`, `value`, `dict_type`, `status`, `color_type`) 
VALUES (28, 2, '关闭', '1', 'common_status', 0, 'info');
```

#### 4.2.2 系统字典的使用场景

| 场景 | 说明 | 示例 |
|-----|------|------|
| **前端下拉框** | 动态展示可选项 | 订单状态、支付方式 |
| **运行时可配置** | 无需重新部署即可修改 | 节假日类型、地区编码 |
| **多语言/国际化** | 标签可动态修改 | 状态名称可调整 |
| **颜色/样式配置** | 前端展示样式 | `color_type`、`css_class` |
| **Excel 导入导出** | 值与标签互转 | 导出时显示"开启"而非"0" |

### 4.3 边界对比

| 对比维度 | 后端枚举 (Java Enum) | 系统字典 (Database Dict) |
|---------|---------------------|-------------------------|
| **定义位置** | Java 源代码中 | 数据库 `system_dict_data` 表 |
| **修改方式** | 修改代码 + 重新编译部署 | 修改数据库数据（立即生效） |
| **值类型** | 类型安全（Integer/Long/String） | 统一 `String` 类型 |
| **编译期检查** | 有（类型错误编译失败） | 无（运行时发现） |
| **业务逻辑使用** | 推荐（类型安全、可重构） | 不推荐（需解析字符串） |
| **前端展示使用** | 需要额外转换 | 直接可用（前端缓存） |
| **适用场景** | 核心业务状态、不可变选项 | 可配置选项、前端展示 |
| **性能** | 极高（JVM 常量池） | 需查询数据库/缓存 |

### 4.4 实际项目中的混合使用

项目中经常同时使用枚举和字典，例如 `UserRespVO.status`：

```java
@Schema(description = "状态，参见 CommonStatusEnum 枚举类", example = "1")
@ExcelProperty(value = "帐号状态", converter = DictConvert.class)
@DictFormat(DictTypeConstants.COMMON_STATUS)
private Integer status;
```

这里 `status` 字段：
- **后端逻辑**: 使用 `CommonStatusEnum` 枚举进行判断
- **Excel 导出**: 使用 `common_status` 字典进行标签翻译
- **前端展示**: 从字典缓存获取下拉选项

---

## 5. 前端字典数据来源

### 5.1 后端 API 接口

#### 5.1.1 获取全部字典列表

**接口**: `GET /system/dict-data/simple-list` 或 `GET /system/dict-data/list-all-simple`

**Controller**: `DictDataController.getSimpleDictDataList()`

```java
@GetMapping(value = {"/list-all-simple", "simple-list"})
@Operation(summary = "获得全部字典数据列表", description = "一般用于管理后台缓存字典数据在本地")
public CommonResult<List<DictDataSimpleRespVO>> getSimpleDictDataList() {
    List<DictDataDO> list = dictDataService.getDictDataList(
            CommonStatusEnum.ENABLE.getStatus(), null);
    return success(BeanUtils.toBean(list, DictDataSimpleRespVO.class));
}
```

#### 5.1.2 Service 层

**路径**: `yudao-module-system/src/main/java/cn/iocoder/yudao/module/system/service/dict/DictDataServiceImpl.java`

```java
@Override
public List<DictDataDO> getDictDataList(Integer status, String dictType) {
    List<DictDataDO> list = dictDataMapper.selectListByStatusAndDictType(status, dictType);
    list.sort(COMPARATOR_TYPE_AND_SORT);
    return list;
}
```

### 5.2 前端缓存机制

#### 5.2.1 初始化时机

前端应用启动时，调用 `/system/dict-data/simple-list` 接口获取所有启用状态的字典数据，存入前端全局状态（Pinia/Vuex 或 localStorage）。

#### 5.2.2 数据结构

```typescript
// 前端字典缓存结构（伪代码）
interface DictData {
  label: string;      // 字典标签
  value: string;      // 字典值（注意：是 string 类型）
  dictType: string;   // 字典类型
  colorType?: string; // 颜色类型
  cssClass?: string;  // CSS 类
}

// 按 dictType 分组缓存
const dictCache: Record<string, DictData[]> = {
  'common_status': [
    { label: '开启', value: '0', dictType: 'common_status', colorType: 'primary' },
    { label: '关闭', value: '1', dictType: 'common_status', colorType: 'info' }
  ],
  'system_user_sex': [
    { label: '男', value: '1', dictType: 'system_user_sex', colorType: 'primary' },
    { label: '女', value: '2', dictType: 'system_user_sex', colorType: 'success' }
  ]
};
```

### 5.3 前端组件绑定

#### 5.3.1 Select 组件

```vue
<template>
  <el-select v-model="form.status" placeholder="请选择状态">
    <el-option
      v-for="item in dictOptions"
      :key="item.value"
      :label="item.label"
      :value="item.value"  <!-- item.value 是 string 类型！ -->
    />
  </el-select>
</template>

<script setup>
import { computed } from 'vue';

// 从全局字典缓存获取
const dictStore = useDictStore();

const dictOptions = computed(() => 
  dictStore.getDictByType('common_status')
);
</script>
```

#### 5.3.2 useDict (若存在)

前端通常会封装一个 `useDict` composable：

```typescript
// 伪代码：useDict.ts
export function useDict(dictTypes: string[]) {
  const dictStore = useDictStore();
  
  const dicts = ref<Record<string, DictData[]>>({});
  
  onMounted(() => {
    dictTypes.forEach(type => {
      dicts.value[type] = dictStore.getDictByType(type);
    });
  });
  
  return dicts;
}

// 使用
const { common_status } = useDict(['common_status']);
```

---

## 6. 数字字符串类型不一致导致回显失败的原因

### 6.1 问题根源

#### 6.1.1 类型链路分析

```
数据库字段: tinyint (0, 1)
    ↓
后端 DO: Integer (0, 1)
    ↓
后端 VO: Integer (0, 1)  -->  JSON 序列化: number (0, 1)
    ↓
前端接收: number (0, 1)
    ↓
前端字典缓存的 value: string ('0', '1')
    ↓
组件 v-model 绑定: number vs string 比较失败
    ↓
回显失败！
```

#### 6.1.2 具体代码示例

**后端返回数据** (JSON):
```json
{
  "id": 1,
  "username": "admin",
  "status": 0,    // number 类型（来自 tinyint）
  "createTime": "2024-01-01 12:00:00"
}
```

**前端字典数据**:
```javascript
const common_status = [
  { label: '开启', value: '0' },  // string 类型
  { label: '关闭', value: '1' }   // string 类型
];
```

**前端组件绑定**:
```vue
<el-select v-model="form.status">  <!-- form.status = 0 (number) -->
  <el-option :value="'0'" label="开启" />  <!-- option.value = '0' (string) -->
  <el-option :value="'1'" label="关闭" />
</el-select>
```

**比较结果**:
```javascript
0 === '0'  // false (严格相等)
0 == '0'   // true (松散相等，但部分组件使用 ===)
```

### 6.2 回显失败的表现

1. **Select 组件**: 显示为空或显示 `value` 而非 `label`
2. **Radio 组件**: 没有选中任何选项
3. **Tag 组件**: 无法匹配颜色类型，显示默认样式
4. **编辑表单**: 编辑时原有值无法正确选中

### 6.3 类型不一致的原因汇总

| 层级 | 位置 | 实际类型 | 期望类型 | 问题 |
|-----|------|---------|---------|------|
| 数据库 | `system_dict_data.value` | `varchar(100)` | - | 存储为字符串 |
| 数据库 | 业务表 `status` 字段 | `tinyint` | - | 存储为整数 |
| 后端 | `DictDataDO.value` | `String` | `String` | 保持一致 |
| 后端 | 业务 `DO.status` | `Integer` | `Integer` | 保持一致 |
| JSON 传输 | `DictDataRespDTO.value` | 字符串 `"0"` | - | 保持一致 |
| JSON 传输 | 业务 `status` | 数字 `0` | - | 保持一致 |
| 前端比较 | `form.status` | `number` | `string` | **不一致！** |

### 6.4 DictFrameworkUtils 中的类型兼容

**路径**: `yudao-framework/yudao-spring-boot-starter-excel/src/main/java/cn/iocoder/yudao/framework/dict/core/DictFrameworkUtils.java`

```java
public static String parseDictDataLabel(String dictType, Integer value) {
    if (value == null) {
        return null;
    }
    // Integer → String 转换
    return parseDictDataLabel(dictType, String.valueOf(value));
}

public static String parseDictDataLabel(String dictType, String value) {
    List<DictDataRespDTO> dictDatas = GET_DICT_DATA_CACHE.get(dictType);
    DictDataRespDTO dictData = CollUtil.findOne(dictDatas, 
        data -> Objects.equals(data.getValue(), value));
    return dictData != null ? dictData.getLabel(): null;
}
```

**关键点**：后端提供了 `Integer` 和 `String` 两个重载方法，通过 `String.valueOf(value)` 解决类型问题。但前端没有这样的处理机制。

---

## 7. 字典更新后缓存何时刷新

### 7.1 后端缓存机制

#### 7.1.1 缓存配置

**路径**: `yudao-framework/yudao-spring-boot-starter-excel/src/main/java/cn/iocoder/yudao/framework/dict/core/DictFrameworkUtils.java`

```java
public class DictFrameworkUtils {
    private static DictDataCommonApi dictDataApi;

    /**
     * 针对 dictType 的字段数据缓存
     */
    private static final LoadingCache<String, List<DictDataRespDTO>> GET_DICT_DATA_CACHE = 
        CacheUtils.buildAsyncReloadingCache(
            Duration.ofMinutes(1L), // 过期时间 1 分钟
            new CacheLoader<String, List<DictDataRespDTO>>() {
                @Override
                public List<DictDataRespDTO> load(String dictType) {
                    return dictDataApi.getDictDataList(dictType);
                }
            });

    public static void init(DictDataCommonApi dictDataApi) {
        DictFrameworkUtils.dictDataApi = dictDataApi;
        log.info("[init][初始化 DictFrameworkUtils 成功]");
    }

    public static void clearCache() {
        GET_DICT_DATA_CACHE.invalidateAll();
    }
}
```

#### 7.1.2 缓存刷新策略

| 刷新时机 | 触发方式 | 说明 |
|---------|---------|------|
| **定时过期** | 1 分钟自动过期 | 缓存项写入后 1 分钟自动失效 |
| **异步刷新** | `buildAsyncReloadingCache` | 首次访问触发加载，后续异步刷新 |
| **手动清除** | `clearCache()` | 应用内调用清除全部缓存 |

#### 7.1.3 缓存类型

使用的是 **Guava LoadingCache + 异步刷新**：

```java
// CacheUtils.buildAsyncReloadingCache 内部实现（推测）
return CacheBuilder.newBuilder()
    .refreshAfterWrite(duration, TimeUnit.MILLISECONDS)  // 写入后 N 时间刷新
    .build(new CacheLoader<K, V>() {
        @Override
        public V load(K key) throws Exception {
            return loader.load(key);
        }
    });
```

### 7.2 前端缓存刷新

#### 7.2.1 刷新时机

| 刷新时机 | 触发方式 | 说明 |
|---------|---------|------|
| **应用启动** | 页面加载/刷新 | 调用 `/system/dict-data/simple-list` |
| **用户登录** | 登录成功后 | 重新获取字典数据 |
| **手动刷新** | 点击刷新按钮 | 部分项目提供刷新功能 |

#### 7.2.2 潜在问题

**问题**: 后端字典数据更新后，前端不会自动感知。

**影响**:
1. 管理员新增字典项后，其他用户前端仍显示旧数据
2. 需刷新页面或重新登录才能获取最新字典

**解决方案**:
1. **WebSocket 推送**: 字典更新时主动通知前端
2. **轮询机制**: 定时检查字典版本号
3. **版本号对比**: 接口返回字典版本，前端对比后决定是否刷新

### 7.3 缓存刷新流程图

```
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│  管理员修改字典  │ ───→ │  数据库更新     │ ───→ │  后端缓存1分钟  │
│  (DictData)     │      │  system_dict_*  │      │  后自动刷新     │
└─────────────────┘      └─────────────────┘      └─────────────────┘
                                                         │
                                                         ▼
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│  用户前端显示   │ ←─── │  前端本地缓存   │ ←─── │  需刷新页面或   │
│  (旧数据)       │      │  (未刷新)       │      │  重新登录获取   │
└─────────────────┘      └─────────────────┘      └─────────────────┘
```

---

## 8. Bug 构造与修复：status 字段类型不一致

### 8.1 问题场景描述

#### 8.1.1 场景设定

假设我们有一个业务表 `demo_order`，其中 `status` 字段：

```sql
-- 业务表
CREATE TABLE `demo_order` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `order_no` varchar(64) NOT NULL COMMENT '订单号',
  `status` tinyint NOT NULL DEFAULT 0 COMMENT '订单状态（0待支付 1已支付 2已取消）',
  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
);

-- 字典数据
INSERT INTO `system_dict_type` (`id`, `name`, `type`, `status`) 
VALUES (9999, '订单状态', 'demo_order_status', 0);

INSERT INTO `system_dict_data` (`id`, `sort`, `label`, `value`, `dict_type`, `status`, `color_type`) 
VALUES 
(9991, 1, '待支付', '0', 'demo_order_status', 0, 'warning'),
(9992, 2, '已支付', '1', 'demo_order_status', 0, 'success'),
(9993, 3, '已取消', '2', 'demo_order_status', 0, 'info');
```

#### 8.1.2 后端代码

**DemoOrderDO.java**:
```java
@TableName("demo_order")
@Data
public class DemoOrderDO extends BaseDO {
    @TableId
    private Long id;
    private String orderNo;
    private Integer status;  // Integer 类型（对应 tinyint）
}
```

**DemoOrderRespVO.java**:
```java
@Schema(description = "订单 Response VO")
@Data
public class DemoOrderRespVO {
    @Schema(description = "订单编号")
    private Long id;
    
    @Schema(description = "订单号")
    private String orderNo;
    
    @Schema(description = "订单状态", example = "0")
    @ExcelProperty(value = "订单状态", converter = DictConvert.class)
    @DictFormat("demo_order_status")
    private Integer status;  // Integer 类型
}
```

#### 8.1.3 前端代码

```vue
<template>
  <div class="order-form">
    <!-- 编辑表单 -->
    <el-form :model="form">
      <el-form-item label="订单状态">
        <el-select v-model="form.status" placeholder="请选择">
          <el-option
            v-for="item in orderStatusOptions"
            :key="item.value"
            :label="item.label"
            :value="item.value"  <!-- item.value 是 string '0','1','2' -->
          />
        </el-select>
      </el-form-item>
    </el-form>
    
    <!-- 列表展示 -->
    <el-table :data="tableData">
      <el-table-column label="订单号" prop="orderNo" />
      <el-table-column label="状态">
        <template #default="scope">
          <el-tag :type="getTagType(scope.row.status)">
            {{ getStatusLabel(scope.row.status) }}
          </el-tag>
        </template>
      </el-table-column>
    </el-table>
  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted } from 'vue';

// 模拟从 API 获取订单数据
const tableData = ref([]);
const form = reactive({
  id: null,
  orderNo: '',
  status: 0  // 默认 number 类型
});

// 模拟从字典缓存获取的数据（value 是 string）
const orderStatusOptions = computed(() => [
  { label: '待支付', value: '0', colorType: 'warning' },
  { label: '已支付', value: '1', colorType: 'success' },
  { label: '已取消', value: '2', colorType: 'info' }
]);

// 获取状态标签
function getStatusLabel(status) {
  // status 是 number (0,1,2)，dict.value 是 string ('0','1','2')
  const dict = orderStatusOptions.value.find(item => item.value === status);
  return dict ? dict.label : '未知';
}

// 获取标签类型
function getTagType(status) {
  const dict = orderStatusOptions.value.find(item => item.value === status);
  return dict ? dict.colorType : 'info';
}

// 模拟加载订单
async function loadOrder(id) {
  // 模拟 API 返回：status 是 number 类型（来自后端 Integer → JSON number）
  const mockApiResponse = {
    id: 1,
    orderNo: 'ORD2024001',
    status: 1  // number 类型（注意：不是 '1'）
  };
  
  Object.assign(form, mockApiResponse);
}

onMounted(() => {
  tableData.value = [
    { id: 1, orderNo: 'ORD2024001', status: 1 },  // number
    { id: 2, orderNo: 'ORD2024002', status: 0 },  // number
  ];
});
</script>
```

### 8.2 Bug 复现

#### 8.2.1 现象 1：列表标签显示异常

```javascript
// tableData[0].status = 1 (number)
// orderStatusOptions[1].value = '1' (string)

getStatusLabel(1) {
  // find 内部比较: '1' === 1 → false
  const dict = orderStatusOptions.value.find(item => item.value === status);
  return dict ? dict.label : '未知';  // 返回 '未知'
}
```

**结果**: 表格中状态显示为"未知"，无法正确显示"已支付"。

#### 8.2.2 现象 2：编辑表单回显失败

```javascript
// 加载订单后
form.status = 1;  // number

// el-select 比较 option.value
// option.value = '1' (string)
// v-model value = 1 (number)
// 1 === '1' → false
```

**结果**: 下拉框显示为空，没有选中"已支付"选项。

#### 8.2.3 现象 3：Tag 颜色异常

```javascript
getTagType(1) {
  // 同样比较失败
  const dict = orderStatusOptions.value.find(item => item.value === status);
  return dict ? dict.colorType : 'info';  // 返回默认 'info'
}
```

**结果**: 所有状态都显示默认颜色，无法区分。

### 8.3 完整修复方案

#### 8.3.1 方案一：前端统一类型转换（推荐）

在前端字典工具函数中统一处理类型转换。

```typescript
// dictUtils.ts

/**
 * 比较字典值，自动处理类型差异
 */
export function compareDictValue(dictValue: string, fieldValue: any): boolean {
  if (dictValue === null || dictValue === undefined || 
      fieldValue === null || fieldValue === undefined) {
    return dictValue === fieldValue;
  }
  
  // 转换为字符串后比较
  return String(dictValue) === String(fieldValue);
}

/**
 * 根据值获取字典项
 */
export function getDictItem(dictList: any[], value: any, valueKey = 'value') {
  return dictList.find(item => compareDictValue(item[valueKey], value));
}

/**
 * 根据值获取字典标签
 */
export function getDictLabel(dictList: any[], value: any, 
                             labelKey = 'label', valueKey = 'value'): string {
  const item = getDictItem(dictList, value, valueKey);
  return item ? item[labelKey] : '';
}

/**
 * 格式化字典列表，统一 value 为字符串或数字
 * @param dictList 原始字典列表
 * @param targetType 目标类型: 'string' | 'number' | 'keep'
 */
export function formatDictList(dictList: any[], 
                                targetType: 'string' | 'number' | 'keep' = 'keep') {
  if (targetType === 'keep') return dictList;
  
  return dictList.map(item => ({
    ...item,
    value: targetType === 'string' 
      ? String(item.value) 
      : Number(item.value)
  }));
}
```

**修改后的前端组件**:

```vue
<script setup>
import { getDictLabel, getDictItem } from '@/utils/dictUtils';

// 获取状态标签（修复后）
function getStatusLabel(status) {
  return getDictLabel(orderStatusOptions.value, status);
}

// 获取标签类型（修复后）
function getTagType(status) {
  const dict = getDictItem(orderStatusOptions.value, status);
  return dict?.colorType || 'info';
}
</script>
```

#### 8.3.2 方案二：后端统一返回类型（更彻底）

修改后端 `DictDataRespDTO`，增加数字类型的 value 字段，或者根据业务字段类型自动转换。

**方案 2.1：增加数字类型字段**

```java
@Data
public class DictDataRespDTO {
    private String label;
    private String value;           // 原始字符串值
    private Integer valueInt;       // 新增：数字值（可能为 null）
    private String dictType;
    private Integer status;
    
    // 或者使用泛型
    private Object valueObj;        // 但 JSON 序列化可能丢失类型信息
}
```

**方案 2.2：后端根据字典类型推断返回类型**

在 Controller 层增加类型推断逻辑（不推荐，增加复杂度）。

#### 8.3.3 方案三：前端字典缓存时转换类型

在获取字典数据后，根据业务需求统一转换 value 的类型。

```typescript
// dictStore.ts (Pinia store)
import { defineStore } from 'pinia';

export const useDictStore = defineStore('dict', {
  state: () => ({
    dictMap: {} as Record<string, any[]>,
    // 配置哪些字典需要转换为数字类型
    numericDictTypes: new Set([
      'common_status',
      'system_user_sex',
      'demo_order_status',
      // ...
    ])
  }),
  
  actions: {
    async loadDicts() {
      const { data } = await getSimpleDictDataList();
      
      // 按 dictType 分组，并根据配置转换类型
      const grouped: Record<string, any[]> = {};
      
      data.forEach(item => {
        if (!grouped[item.dictType]) {
          grouped[item.dictType] = [];
        }
        
        // 根据配置转换 value 类型
        const processedItem = { ...item };
        if (this.numericDictTypes.has(item.dictType)) {
          processedItem.value = Number(item.value);
        }
        
        grouped[item.dictType].push(processedItem);
      });
      
      this.dictMap = grouped;
    },
    
    getDictByType(type: string) {
      return this.dictMap[type] || [];
    }
  }
});
```

#### 8.3.4 方案四：使用自定义指令/组件封装

封装通用的字典组件，内部处理类型转换。

```vue
<!-- DictSelect.vue -->
<template>
  <el-select v-model="modelValue" :placeholder="placeholder" v-bind="$attrs">
    <el-option
      v-for="item in processedOptions"
      :key="item.value"
      :label="item.label"
      :value="item.value"
    />
  </el-select>
</template>

<script setup lang="ts">
import { computed } from 'vue';

const props = defineProps<{
  modelValue: any;
  options: any[];
  valueType?: 'string' | 'number' | 'auto';
  valueKey?: string;
  labelKey?: string;
}>();

const emit = defineEmits<{
  (e: 'update:modelValue', value: any): void;
}>();

// 处理后的 options，统一 value 类型
const processedOptions = computed(() => {
  if (props.valueType === 'keep' || !props.valueType) {
    return props.options;
  }
  
  return props.options.map(item => {
    let value = item[props.valueKey || 'value'];
    
    if (props.valueType === 'string') {
      value = String(value);
    } else if (props.valueType === 'number') {
      value = Number(value);
    }
    // 'auto' 模式：尝试保持原类型
    else if (props.valueType === 'auto') {
      // 如果 v-model 是数字，尝试转换
      if (typeof props.modelValue === 'number') {
        value = Number(value);
      }
    }
    
    return {
      ...item,
      [props.valueKey || 'value']: value
    };
  });
});
</script>
```

**使用方式**:

```vue
<template>
  <DictSelect
    v-model="form.status"
    :options="orderStatusOptions"
    value-type="number"  <!-- 统一转为 number -->
    placeholder="请选择状态"
  />
</template>
```

### 8.4 推荐修复方案

#### 8.4.1 推荐：方案一 + 方案三结合

1. **基础层**: 实现 `dictUtils.ts` 提供类型安全的比较工具函数
2. **缓存层**: 在字典 Store 中可配置哪些字典需要类型转换
3. **组件层**: 使用封装的字典组件，自动处理类型

#### 8.4.2 完整修复代码

**1. 字典工具函数 (`src/utils/dict.ts`)**:

```typescript
export interface DictData {
  label: string;
  value: string | number;
  dictType?: string;
  colorType?: string;
  cssClass?: string;
  [key: string]: any;
}

/**
 * 安全比较字典值，自动处理类型差异
 * 使用松散相等，但显式处理 null/undefined
 */
export function isDictValueEqual(dictValue: string | number, 
                                 fieldValue: any): boolean {
  if (dictValue === null || dictValue === undefined ||
      fieldValue === null || fieldValue === undefined) {
    return dictValue === fieldValue;
  }
  
  // 转换为字符串后比较（最稳妥的方式）
  return String(dictValue) === String(fieldValue);
}

/**
 * 从字典列表中查找匹配项
 */
export function findDictItem(dictList: DictData[], 
                             value: any, 
                             valueKey: string = 'value'): DictData | undefined {
  return dictList.find(item => 
    isDictValueEqual(item[valueKey], value)
  );
}

/**
 * 获取字典标签
 */
export function getDictLabel(dictList: DictData[], 
                             value: any,
                             labelKey: string = 'label',
                             valueKey: string = 'value'): string {
  const item = findDictItem(dictList, value, valueKey);
  return item?.[labelKey] ?? '';
}

/**
 * 获取字典颜色类型
 */
export function getDictColor(dictList: DictData[], 
                             value: any,
                             valueKey: string = 'value'): string {
  const item = findDictItem(dictList, value, valueKey);
  return item?.colorType ?? 'info';
}

/**
 * 格式化字典列表，统一 value 类型
 */
export function formatDictValues<T extends DictData>(
  dictList: T[],
  targetType: 'string' | 'number' | ((item: T) => string | number)
): T[] {
  return dictList.map(item => {
    let newValue: string | number;
    
    if (typeof targetType === 'function') {
      newValue = targetType(item);
    } else if (targetType === 'string') {
      newValue = String(item.value);
    } else {
      newValue = Number(item.value);
    }
    
    return { ...item, value: newValue };
  });
}
```

**2. 使用示例**:

```vue
<template>
  <div>
    <!-- 列表展示 -->
    <el-table :data="tableData">
      <el-table-column label="订单号" prop="orderNo" />
      <el-table-column label="状态" width="120">
        <template #default="scope">
          <el-tag :type="getDictColor(orderStatusOptions, scope.row.status)">
            {{ getDictLabel(orderStatusOptions, scope.row.status) }}
          </el-tag>
        </template>
      </el-table-column>
    </el-table>
    
    <!-- 编辑表单 -->
    <el-form :model="form">
      <el-form-item label="订单状态">
        <el-select v-model="form.status" placeholder="请选择">
          <el-option
            v-for="item in orderStatusOptions"
            :key="String(item.value)"  <!-- key 也用 String 避免问题 -->
            :label="item.label"
            :value="item.value"
          />
        </el-select>
      </el-form-item>
    </el-form>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, computed } from 'vue';
import { getDictLabel, getDictColor, formatDictValues } from '@/utils/dict';

// 字典数据（从后端获取，value 是 string）
const rawOrderStatusOptions = [
  { label: '待支付', value: '0', colorType: 'warning' },
  { label: '已支付', value: '1', colorType: 'success' },
  { label: '已取消', value: '2', colorType: 'info' }
];

// 方案 A：在使用时用工具函数比较（推荐，最灵活）
const orderStatusOptions = computed(() => rawOrderStatusOptions);

// 方案 B：直接转换为 number 类型（如果确定都是数字字典）
// const orderStatusOptions = computed(() => 
//   formatDictValues(rawOrderStatusOptions, 'number')
// );

// 表单数据（status 是 number 类型，来自后端）
const form = reactive({
  id: null,
  orderNo: '',
  status: 0  // number
});

const tableData = ref([
  { id: 1, orderNo: 'ORD2024001', status: 1 },  // number
  { id: 2, orderNo: 'ORD2024002', status: 0 },  // number
]);
</script>
```

**3. 关键修复点总结**:

| 问题点 | 修复方式 |
|-------|---------|
| 列表标签显示 | 使用 `getDictLabel()` 替代 `===` 比较 |
| Tag 颜色显示 | 使用 `getDictColor()` 替代直接查找 |
| 下拉框回显 | 确保 v-model 与 option.value 类型一致，或使用 `:value="Number(item.value)"` |
| 条件判断 | 使用 `isDictValueEqual()` 替代 `===` |

### 8.5 修复验证

#### 8.5.1 验证清单

- [ ] 列表中状态标签正确显示（"已支付"而非"未知"）
- [ ] 列表中 Tag 颜色正确（success/warning/info）
- [ ] 编辑表单加载后下拉框正确选中
- [ ] 选择新值后保存正常
- [ ] Excel 导出显示正确标签
- [ ] Excel 导入后值正确转换

#### 8.5.2 测试用例

```javascript
// 测试 isDictValueEqual
console.log(isDictValueEqual('0', 0));      // true ✓
console.log(isDictValueEqual('1', 1));      // true ✓
console.log(isDictValueEqual(0, 0));        // true ✓
console.log(isDictValueEqual('0', '0'));    // true ✓
console.log(isDictValueEqual('0', 1));      // false ✓
console.log(isDictValueEqual('0', null));   // false ✓
console.log(isDictValueEqual(null, null));  // true ✓

// 测试 getDictLabel
const dictList = [
  { label: '开启', value: '0', colorType: 'primary' },
  { label: '关闭', value: '1', colorType: 'info' }
];

console.log(getDictLabel(dictList, 0));   // '开启' ✓
console.log(getDictLabel(dictList, '0')); // '开启' ✓
console.log(getDictLabel(dictList, 1));   // '关闭' ✓
console.log(getDictLabel(dictList, 2));   // '' ✓
```

---

## 9. 完整数据流总结

### 9.1 字典数据完整链路

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        数据库层                                          │
│  system_dict_type (字典类型定义)                                          │
│  system_dict_data (字典数据: value = varchar 字符串)                      │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        后端实体层                                        │
│  DictTypeDO.java (type = String)                                        │
│  DictDataDO.java (value = String)  ←── 这里是 String                     │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        后端 Service 层                                   │
│  DictFrameworkUtils (Guava Cache, 1分钟过期)                              │
│  - GET_DICT_DATA_CACHE: LoadingCache<String, List<DictDataRespDTO>>     │
│  - parseDictDataLabel(Integer/String) 重载方法处理类型                     │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        后端 API 层                                       │
│  GET /system/dict-data/simple-list                                      │
│  返回: List<DictDataSimpleRespVO>                                        │
│  JSON: [{ "label": "开启", "value": "0", ... }]  ←── value 是字符串      │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
                                   ▼ HTTP
┌─────────────────────────────────────────────────────────────────────────┐
│                        前端应用层                                        │
│  1. 应用启动时调用 API 获取所有字典                                        │
│  2. 存入前端状态管理 (Pinia/Vuex) 或 localStorage                         │
│  3. 按 dictType 分组缓存                                                  │
│                                                                         │
│  前端字典数据:                                                           │
│  {                                                                       │
│    'common_status': [                                                    │
│      { label: '开启', value: '0' },  // string                           │
│      { label: '关闭', value: '1' }   // string                           │
│    ]                                                                     │
│  }                                                                       │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        前端组件绑定                                       │
│                                                                         │
│  业务数据 (从后端获取):                                                    │
│  { id: 1, username: 'admin', status: 0 }  ←── status 是 number          │
│                                                                         │
│  组件使用:                                                               │
│  <el-select v-model="form.status">  <!-- form.status = 0 (number) -->   │
│    <el-option :value="'0'" />       <!-- option.value = '0' (string) -->│
│  </el-select>                                                           │
│                                                                         │
│  ⚠️ 类型不一致问题: 0 === '0' → false (严格相等)                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.2 业务数据完整链路

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        业务数据库表                                       │
│  demo_order                                                              │
│  ├── id: bigint                                                          │
│  ├── order_no: varchar                                                   │
│  └── status: tinyint (0, 1, 2)  ←── 整数类型                              │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        后端业务实体                                      │
│  DemoOrderDO.java                                                        │
│  ├── id: Long                                                            │
│  ├── orderNo: String                                                     │
│  └── status: Integer  ←── 对应 tinyint，Integer 类型                      │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        后端 VO                                           │
│  DemoOrderRespVO.java                                                    │
│  ├── id: Long                                                            │
│  ├── orderNo: String                                                     │
│  └── status: Integer                                                     │
│      @DictFormat("demo_order_status")  // Excel 导出时用字典翻译           │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        JSON 传输                                         │
│  {                                                                       │
│    "id": 1,                                                              │
│    "orderNo": "ORD2024001",                                              │
│    "status": 0  ←── JSON number 类型（Integer → number）                  │
│  }                                                                       │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        前端接收                                          │
│  form.status = 0  ←── number 类型                                        │
│                                                                         │
│  字典数据（来自 API）:                                                     │
│  orderStatusOptions = [                                                  │
│    { label: '待支付', value: '0' },  ←── string 类型                     │
│    { label: '已支付', value: '1' },  ←── string 类型                     │
│    { label: '已取消', value: '2' }   ←── string 类型                     │
│  ]                                                                       │
│                                                                         │
│  ⚠️ 类型不匹配！                                                          │
│  form.status = 0      (number)                                           │
│  option.value = '0'  (string)                                            │
│  0 === '0' → false (严格相等)                                             │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.3 类型不一致问题总结

| 数据类型 | 业务数据 (status) | 字典数据 (value) | 比较结果 |
|---------|------------------|-----------------|---------|
| 数据库 | `tinyint` (0) | `varchar` ('0') | N/A |
| 后端实体 | `Integer` (0) | `String` ("0") | 后端有重载处理 |
| JSON 传输 | `number` (0) | `string` ("0") | N/A |
| 前端变量 | `number` (0) | `string` ("0") | `0 === '0'` → **false** ❌ |

### 9.4 解决方案总结

1. **后端**: 已通过 `DictFrameworkUtils.parseDictDataLabel(Integer/String)` 重载处理
2. **前端**: 需自行实现类型安全的比较工具函数
3. **推荐**: 使用 `String(a) === String(b)` 进行比较，或统一转换类型

---

## 10. 最佳实践建议

### 10.1 字典设计建议

1. **统一 value 类型**: 
   - 数字类型的字典，建议在数据库中也使用数字类型（但需要修改 `DictDataDO.value` 类型，改动较大）
   - 或者统一使用字符串，业务字段也用 String 存储（影响查询性能）

2. **字典类型常量**:
   - 在对应模块的 `DictTypeConstants` 中定义字典类型常量
   - 代码生成时指定 `dictType`，便于后续维护

3. **与枚举配合**:
   - 核心业务逻辑使用 Java 枚举（类型安全）
   - 前端展示和 Excel 导出使用系统字典（灵活配置）

### 10.2 前端开发建议

1. **封装字典工具函数**:
   - 始终使用 `getDictLabel()`、`findDictItem()` 等工具函数
   - 避免直接使用 `===` 比较字典值

2. **类型转换策略**:
   - 策略 A：字典 value 保持 String，业务值转为 String 比较
   - 策略 B：字典 value 转为 Number，与业务值保持一致
   - 策略 C：使用 `String(a) === String(b)` 松散比较

3. **组件封装**:
   - 封装 `DictSelect`、`DictRadio`、`DictTag` 等组件
   - 组件内部处理类型转换，业务代码无需关心

### 10.3 缓存管理建议

1. **后端缓存**:
   - 当前 1 分钟过期是合理的
   - 考虑在字典更新时主动清除缓存（调用 `DictFrameworkUtils.clearCache()`）

2. **前端缓存**:
   - 应用启动和登录时刷新
   - 考虑增加手动刷新按钮
   - 重要字典可考虑 WebSocket 实时更新

---

## 11. 参考文件清单

| 文件路径 | 说明 |
|---------|------|
| `yudao-module-system/src/main/java/.../dict/DictTypeDO.java` | 字典类型实体 |
| `yudao-module-system/src/main/java/.../dict/DictDataDO.java` | 字典数据实体 |
| `yudao-module-system/src/main/java/.../dict/DictDataServiceImpl.java` | 字典数据服务 |
| `yudao-framework/.../dict/core/DictFrameworkUtils.java` | 字典框架工具类（含缓存） |
| `yudao-framework/.../excel/core/annotations/DictFormat.java` | 字典格式化注解 |
| `yudao-framework/.../excel/core/convert/DictConvert.java` | 字典转换器 |
| `yudao-module-system/src/main/java/.../enums/DictTypeConstants.java` | 字典类型常量 |
| `yudao-framework/.../enums/CommonStatusEnum.java` | 通用状态枚举 |
| `sql/mysql/ruoyi-vue-pro.sql` | 数据库表结构和初始数据 |
