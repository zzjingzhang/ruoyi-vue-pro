# Ruoyi-Vue-Pro 代码生成器全过程深度分析

## 一、整体架构与流程概述

Ruoyi-Vue-Pro（现名 Yudao）代码生成器是一个基于 Velocity 模板引擎的完整代码生成系统，位于 `yudao-module-infra` 模块下。其核心流程分为三个阶段：

1. **元数据读取阶段**：从数据库读取表结构信息
2. **配置构建阶段**：将数据库元数据转换为代码生成配置
3. **代码生成阶段**：基于 Velocity 模板生成后端代码、前端代码和 SQL 菜单

### 1.1 核心类职责

| 类名 | 文件位置 | 职责 |
|------|---------|------|
| `CodegenService` | `CodegenService.java:19` | 代码生成 Service 接口 |
| `CodegenServiceImpl` | `CodegenServiceImpl.java:47` | 核心业务实现 |
| `CodegenBuilder` | `CodegenBuilder.java:31` | 元数据到配置对象的构建器 |
| `CodegenEngine` | `CodegenEngine.java:60` | Velocity 模板引擎执行器 |
| `DatabaseTableService` | `DatabaseTableServiceImpl.java:27` | 数据库表元数据读取 |
| `CodegenConvert` | `CodegenConvert.java:23` | MapStruct 转换器 |

---

## 二、数据库元数据读取阶段

### 2.1 元数据来源

代码生成器使用 **MyBatis Plus Generator** 来解析数据库表结构。

**入口代码**：`CodegenServiceImpl.java:77-82`
```java
private Long createCodegen(String author, Long dataSourceConfigId, String tableName) {
    // 从数据库中，获得数据库表结构
    TableInfo tableInfo = databaseTableService.getTable(dataSourceConfigId, tableName);
    // 导入
    return createCodegen0(author, dataSourceConfigId, tableInfo);
}
```

**底层实现**：`DatabaseTableServiceImpl.java:46-75`
```java
private List<TableInfo> getTableList0(Long dataSourceConfigId, String name) {
    // 1. 获取数据源配置
    DataSourceConfigDO config = dataSourceConfigService.getDataSourceConfig(dataSourceConfigId);
    
    // 2. 使用 MyBatis Plus Generator 解析表结构
    DataSourceConfig.Builder dataSourceConfigBuilder = 
        new DataSourceConfig.Builder(config.getUrl(), config.getUsername(), config.getPassword());
    
    // 3. 配置全局策略
    GlobalConfig globalConfig = new GlobalConfig.Builder()
        .dateType(DateType.TIME_PACK)  // 只使用 LocalDateTime 类型
        .build();
    
    // 4. 构建配置并获取表信息
    ConfigBuilder builder = new ConfigBuilder(null, dataSourceConfigBuilder.build(), 
        strategyConfig.build(), null, globalConfig, null);
    List<TableInfo> tables = builder.getTableInfoList();
    return tables;
}
```

### 2.2 元数据校验

在导入表之前，会进行严格的校验：`CodegenServiceImpl.java:112-127`
- 表名和字段名不能为空
- 表注释和字段注释不能为空（强制要求）
- 表不能重复导入

---

## 三、配置构建阶段

### 3.1 表配置构建（CodegenTableDO）

**构建逻辑**：`CodegenBuilder.java:100-124`

以 `system_dept` 表为例，默认配置生成规则：

| 属性 | 规则 | 示例值 |
|------|------|--------|
| `moduleName` | 第一个下划线前的部分（小写） | `system` |
| `businessName` | 第一个下划线后的部分（驼峰转小写） | `dept` |
| `className` | 第一个下划线后的部分（驼峰+首字母大写） | `Dept` |
| `classComment` | 表注释去除"表"后缀 | `部门` |
| `templateType` | 默认为单表（ONE = 1） | `1` |

### 3.2 字段配置构建（CodegenColumnDO）

**核心转换逻辑**：`CodegenConvert.java:38-52`

```java
@Mappings({
    @Mapping(source = "name", target = "columnName"),
    @Mapping(source = "metaInfo.jdbcType", target = "dataType", qualifiedByName = "getDataType"),
    @Mapping(source = "comment", target = "columnComment"),
    @Mapping(source = "metaInfo.nullable", target = "nullable"),
    @Mapping(source = "keyFlag", target = "primaryKey"),
    @Mapping(source = "columnType.type", target = "javaType"),  // 关键：MyBatis Plus 自动映射
    @Mapping(source = "propertyName", target = "javaField"),
})
CodegenColumnDO convert(TableField bean);
```

---

## 四、字段类型映射详解

### 4.1 数据库类型 → Java 类型映射

由 MyBatis Plus Generator 自动完成，核心映射关系：

| 数据库类型（JDBC） | Java 类型 | 备注 |
|-------------------|----------|------|
| `BIGINT` | `Long` | - |
| `INTEGER` | `Integer` | - |
| `SMALLINT` | `Short` | - |
| `TINYINT` | `Byte` | **注意：会被转换为 Integer** |
| `VARCHAR` | `String` | - |
| `TEXT` / `LONGVARCHAR` | `String` | - |
| `DATE` / `TIME` / `TIMESTAMP` | `LocalDateTime` | 全局配置：`DateType.TIME_PACK` |
| `DECIMAL` | `BigDecimal` | - |
| `BIT` / `BOOLEAN` | `Boolean` | - |
| `BLOB` / `BINARY` | `byte[]` | - |

**重要特殊处理**：`CodegenBuilder.java:133-136`
```java
// 特殊处理：Byte => Integer
if (Byte.class.getSimpleName().equals(column.getJavaType())) {
    column.setJavaType(Integer.class.getSimpleName());
}
```

**原因**：MySQL 的 `TINYINT` 类型在 MyBatis Plus 中默认映射为 `Byte`，但在实际业务中通常表示状态、类型等枚举值，使用 `Integer` 更方便。

### 4.2 Java 类型 → TypeScript 类型映射

在 Vue 3 API 模板中定义：`api.ts.vm:14-35`

```typescript
#if(${column.javaType.toLowerCase()} == "long" || ${column.javaType.toLowerCase()} == "integer" 
    || ${column.javaType.toLowerCase()} == "short" || ${column.javaType.toLowerCase()} == "double" 
    || ${column.javaType.toLowerCase()} == "bigdecimal")
    ${column.javaField}: number; // 所有数值类型统一为 number
#elseif(${column.javaType.toLowerCase()} == "date" || ${column.javaType.toLowerCase()} == "localdate" 
    || ${column.javaType.toLowerCase()} == "localdatetime")
    ${column.javaField}: string | Dayjs; // 日期类型
#else
    ${column.javaField}: ${column.javaType.toLowerCase()}; // Boolean、String 等
#end
```

**映射规则汇总**：

| Java 类型 | TypeScript 类型 |
|----------|----------------|
| `Long` / `Integer` / `Short` / `Double` / `BigDecimal` | `number` |
| `LocalDateTime` / `LocalDate` / `Date` | `string \| Dayjs` |
| `String` | `string` |
| `Boolean` | `boolean` |

### 4.3 Java 类型 → 表单组件映射

**默认映射规则**：`CodegenBuilder.java:49-183`

#### 4.3.1 基于字段后缀的映射

```java
private static final Map<String, CodegenColumnHtmlTypeEnum> COLUMN_HTML_TYPE_MAPPINGS =
    MapUtil.<String, CodegenColumnHtmlTypeEnum>builder()
        .put("status", CodegenColumnHtmlTypeEnum.RADIO)      // *status → 单选框
        .put("sex", CodegenColumnHtmlTypeEnum.RADIO)         // *sex → 单选框
        .put("type", CodegenColumnHtmlTypeEnum.SELECT)       // *type → 下拉框
        .put("image", CodegenColumnHtmlTypeEnum.IMAGE_UPLOAD)// *image → 图片上传
        .put("file", CodegenColumnHtmlTypeEnum.FILE_UPLOAD)  // *file → 文件上传
        .put("content", CodegenColumnHtmlTypeEnum.EDITOR)    // *content → 富文本
        .put("description", CodegenColumnHtmlTypeEnum.EDITOR)// *description → 富文本
        .put("time", CodegenColumnHtmlTypeEnum.DATETIME)     // *time → 日期控件
        .put("date", CodegenColumnHtmlTypeEnum.DATETIME)     // *date → 日期控件
        .build();
```

#### 4.3.2 基于 Java 类型的映射

```java
// 如果是 Boolean 类型时，设置为 radio 类型
if (Boolean.class.getSimpleName().equals(column.getJavaType())) {
    column.setHtmlType(CodegenColumnHtmlTypeEnum.RADIO.getType());
}
// 如果是 LocalDateTime 类型，则设置为 datetime 类型
if (LocalDateTime.class.getSimpleName().equals(column.getJavaType())) {
    column.setHtmlType(CodegenColumnHtmlTypeEnum.DATETIME.getType());
}
// 兜底，设置默认为 input 类型
if (column.getHtmlType() == null) {
    column.setHtmlType(CodegenColumnHtmlTypeEnum.INPUT.getType());
}
```

#### 4.3.3 完整映射表

| 字段后缀 / 类型 | 表单组件 | 组件值 |
|----------------|---------|--------|
| `*status` / `*sex` | 单选框 | `radio` |
| `*type` | 下拉框 | `select` |
| `*image` | 图片上传 | `imageUpload` |
| `*file` | 文件上传 | `fileUpload` |
| `*content` / `*description` | 富文本编辑器 | `editor` |
| `*time` / `*date` | 日期时间控件 | `datetime` |
| `Boolean` 类型 | 单选框 | `radio` |
| `LocalDateTime` 类型 | 日期时间控件 | `datetime` |
| 其他（默认） | 文本框 | `input` |

#### 4.3.4 列表查询条件映射

```java
private static final Map<String, CodegenColumnListConditionEnum> COLUMN_LIST_OPERATION_CONDITION_MAPPINGS =
    MapUtil.<String, CodegenColumnListConditionEnum>builder()
        .put("name", CodegenColumnListConditionEnum.LIKE)      // *name → 模糊查询
        .put("time", CodegenColumnListConditionEnum.BETWEEN)   // *time → 范围查询
        .put("date", CodegenColumnListConditionEnum.BETWEEN)   // *date → 范围查询
        .build();
// 默认使用 = 精确查询
```

---

## 五、枚举/字典字段识别与处理

### 5.1 字典字段配置

字典字段通过 `CodegenColumnDO.dictType` 属性标识。该属性**不是自动从数据库读取的**，而是需要在代码生成器界面手动配置。

**数据结构**：`CodegenColumnDO.java:93-94`
```java
/**
 * 字典类型
 * 关联 DictTypeDO 的 type 属性
 */
private String dictType;
```

### 5.2 字典字段在后端的处理

#### 5.2.1 DO 类中的处理：`do.vm:57-74`

```java
#if ("$!column.dictType" != "")
    /**
     * ${column.columnComment}
     * 枚举 {@link TODO ${column.dictType} 对应的类}
     */
    #if ($voType == 20)
    @ExcelProperty(value = "${column.columnComment}", converter = DictConvert.class)
    @DictFormat("${column.dictType}")  // 字典格式化注解
    #end
    private ${column.javaType} ${column.javaField};
#end
```

#### 5.2.2 RespVO 类中的处理：`respVO.vm:42-49`

```java
#if ("$!column.dictType" != "")
    @ExcelProperty(value = "${column.columnComment}", converter = DictConvert.class)
    @DictFormat("${column.dictType}")
#else
    @ExcelProperty("${column.columnComment}")
#end
```

### 5.3 字典字段在前端的处理

#### 5.3.1 字典方法选择：`index.vue.vm:18-25`

```vue
#set ($dictMethod = "getDictOptions")
#if ($javaType == "Integer" || $javaType == "Long" || $javaType == "Byte" || $javaType == "Short")
    #set ($dictMethod = "getIntDictOptions")
#elseif ($javaType == "String")
    #set ($dictMethod = "getStrDictOptions")
#elseif ($javaType == "Boolean")
    #set ($dictMethod = "getBoolDictOptions")
#end
```

#### 5.3.2 表单中的字典渲染：`index.vue.vm:36-55`

```vue
#if ("" != $dictType)
<el-select v-model="queryParams.${javaField}">
  <el-option
    v-for="dict in $dictMethod(DICT_TYPE.$dictType.toUpperCase())"
    :key="dict.value"
    :label="dict.label"
    :value="dict.value"
  />
</el-select>
#else
<el-select v-model="queryParams.${javaField}">
  <el-option label="请选择字典生成" value="" />
</el-select>
#end
```

---

## 六、主子表与树表的特殊处理

### 6.1 模板类型枚举：`CodegenTemplateTypeEnum.java`

```java
public enum CodegenTemplateTypeEnum {
    ONE(1),           // 单表
    TREE(2),          // 树表
    MASTER_NORMAL(10),// 主子表 - 主表 - 普通模式
    MASTER_ERP(11),   // 主子表 - 主表 - ERP 模式
    MASTER_INNER(12), // 主子表 - 主表 - 内嵌模式
    SUB(15),          // 主子表 - 子表
}
```

### 6.2 主子表配置

**主表配置字段**：`CodegenTableDO.java:118-154`

| 字段 | 说明 |
|------|------|
| `masterTableId` | 主表 ID（子表使用） |
| `subJoinColumnId` | 子表关联主表的字段 ID |
| `subJoinMany` | 是否一对多关系 |

**主子表生成逻辑**：`CodegenEngine.java:273-297`

```java
// 如果是主子表，则加载对应的子表信息
List<CodegenTableDO> subTables = null;
List<List<CodegenColumnDO>> subColumnsList = null;
if (CodegenTemplateTypeEnum.isMaster(table.getTemplateType())) {
    // 校验子表存在
    subTables = codegenTableMapper.selectListByTemplateTypeAndMasterTableId(
        CodegenTemplateTypeEnum.SUB.getType(), tableId);
    // 校验子表的关联字段存在
    subColumnsList = new ArrayList<>();
    for (CodegenTableDO subTable : subTables) {
        List<CodegenColumnDO> subColumns = codegenColumnMapper.selectListByTableId(subTable.getId());
        subColumnsList.add(subColumns);
    }
}
```

**主子表模板匹配**：`CodegenEngine.java:380-407`

```java
// 主子表的模式匹配。目的：过滤掉个性化的模版
if (vmPath.contains("_normal")
    && ObjectUtil.notEqual(table.getTemplateType(), CodegenTemplateTypeEnum.MASTER_NORMAL.getType())) {
    return; // 不是普通模式，跳过普通模式模板
}
if (vmPath.contains("_erp")
    && ObjectUtil.notEqual(table.getTemplateType(), CodegenTemplateTypeEnum.MASTER_ERP.getType())) {
    return; // 不是 ERP 模式，跳过 ERP 模式模板
}
// 逐个生成子表代码
for (int i = 0; i < subTables.size(); i++) {
    bindingMap.put("subIndex", i);
    generateCode(result, vmPath, filePath, bindingMap);
}
```

### 6.3 树表配置

**树表配置字段**：`CodegenTableDO.java:139-154`

| 字段 | 说明 |
|------|------|
| `treeParentColumnId` | 父节点字段 ID |
| `treeNameColumnId` | 节点名称字段 ID |

**树表生成逻辑**：`CodegenEngine.java:473-482`

```java
// 特殊：树表专属逻辑
if (CodegenTemplateTypeEnum.isTree(table.getTemplateType())) {
    CodegenColumnDO treeParentColumn = CollUtil.findOne(columns,
        column -> Objects.equals(column.getId(), table.getTreeParentColumnId()));
    bindingMap.put("treeParentColumn", treeParentColumn);
    CodegenColumnDO treeNameColumn = CollUtil.findOne(columns,
        column -> Objects.equals(column.getId(), table.getTreeNameColumnId()));
    bindingMap.put("treeNameColumn", treeNameColumn);
}
```

**树表 DO 特殊处理**：`do.vm:49-52`

```java
#if ( $table.templateType == 2 )
    public static final Long ${treeParentColumn_javaField_underlineCase.toUpperCase()}_ROOT = 0L;
#end
```

**树表 API 特殊处理**：`api.ts.vm:58-68`

```typescript
#if ( $table.templateType != 2 )
  // 非树表：分页查询
  get${simpleClassName}Page: async (params: any) => {
    return await request.get({ url: `${baseURL}/page`, params })
  },
#else
  // 树表：列表查询（不分页）
  get${simpleClassName}List: async (params) => {
    return await request.get({ url: `${baseURL}/list`, params })
  },
#end
```

**树表 VO 特殊处理**：`api.ts.vm:39-41`

```typescript
#if ( $table.templateType == 2 )
    children?: ${simpleClassName}[];
#end
```

---

## 七、权限标识生成与匹配

### 7.1 权限标识格式

**权限前缀生成**：`CodegenEngine.java:470`

```java
// permission 前缀
bindingMap.put("permissionPrefix", 
    table.getModuleName() + ":" + simpleClassNameStrikeCase);
```

**格式**：`moduleName:simpleClassName-strikeCase`

**示例**：
- 表名 `system_user` → `moduleName=system`, `className=User`, `simpleClassNameStrikeCase=user`
- 权限前缀：`system:user`

### 7.2 权限操作类型

在 SQL 模板中定义：`sql.vm:2-8`

```java
#if ($importEnable)
#set ($functionNames = ['查询', '创建', '更新', '删除', '导出', '导入'])
#set ($functionOps = ['query', 'create', 'update', 'delete', 'export', 'import'])
#else
#set ($functionNames = ['查询', '创建', '更新', '删除', '导出'])
#set ($functionOps = ['query', 'create', 'update', 'delete', 'export'])
#end
```

### 7.3 后端 @PreAuthorize 注解

**Controller 模板**：`controller.vm:50-78`

```java
@PostMapping("/create")
@Operation(summary = "创建${table.classComment}")
#if ($sceneEnum.scene == 1)
@PreAuthorize("@ss.hasPermission('${permissionPrefix}:create')")
#end
public CommonResult<${primaryColumn.javaType}> create${simpleClassName}(...) { ... }

@PutMapping("/update")
@Operation(summary = "更新${table.classComment}")
#if ($sceneEnum.scene == 1)
@PreAuthorize("@ss.hasPermission('${permissionPrefix}:update')")
#end
public CommonResult<Boolean> update${simpleClassName}(...) { ... }

@DeleteMapping("/delete")
@Operation(summary = "删除${table.classComment}")
#if ($sceneEnum.scene == 1)
@PreAuthorize("@ss.hasPermission('${permissionPrefix}:delete')")
#end
public CommonResult<Boolean> delete${simpleClassName}(...) { ... }

@GetMapping("/get")
@Operation(summary = "获得${table.classComment}")
#if ($sceneEnum.scene == 1)
@PreAuthorize("@ss.hasPermission('${permissionPrefix}:query')")
#end
public CommonResult<${respVOClass}> get${simpleClassName}(...) { ... }

@GetMapping("/page")
@Operation(summary = "获得${table.classComment}分页")
#if ($sceneEnum.scene == 1)
@PreAuthorize("@ss.hasPermission('${permissionPrefix}:query')")
#end
public CommonResult<PageResult<${respVOClass}>> get${simpleClassName}Page(...) { ... }

@GetMapping("/export-excel")
@Operation(summary = "导出${table.classComment} Excel")
#if ($sceneEnum.scene == 1)
@PreAuthorize("@ss.hasPermission('${permissionPrefix}:export')")
#end
public void export${simpleClassName}Excel(...) { ... }
```

### 7.4 前端按钮权限控制

**Vue 3 模板**：`index.vue.vm:88-129`

```vue
<el-button type="primary" plain @click="openForm('create')"
  v-hasPermi="['${permissionPrefix}:create']">
  新增
</el-button>

<el-button type="warning" plain @click="handleImport"
  v-hasPermi="['${permissionPrefix}:import']">
  导入
</el-button>

<el-button type="success" plain @click="handleExport"
  v-hasPermi="['${permissionPrefix}:export']">
  导出
</el-button>

<el-button type="danger" plain @click="handleDeleteBatch"
  v-hasPermi="['${table.moduleName}:${simpleClassName_strikeCase}:delete']">
  批量删除
</el-button>
```

### 7.5 SQL 菜单权限

**SQL 模板**：`sql.vm:27-51`

```sql
-- 菜单 SQL
INSERT INTO system_menu(
    name, permission, type, sort, parent_id,
    path, icon, component, status, component_name
)
VALUES (
    '${table.classComment}管理', '', 2, 0, ${table.parentMenuId},
    '${simpleClassName_strikeCase}', '', '${table.moduleName}/${table.businessName}/index', 0, '${table.className}'
);

-- 按钮 SQL
#foreach ($functionName in $functionNames)
#set ($index = $foreach.count - 1)
INSERT INTO system_menu(
    name, permission, type, sort, parent_id,
    path, icon, component, status
)
VALUES (
    '${table.classComment}${functionName}', '${permissionPrefix}:${functionOps.get($index)}', 3, $foreach.count, @parentId,
    '', '', '', 0
);
#end
```

### 7.6 权限匹配完整链路

```
数据库表 system_user
    ↓
代码生成器配置
    ↓
权限前缀: system:user
    ↓
┌─────────────────────────────────────────────────────────┐
│                    三方匹配                              │
├─────────────────────────────────────────────────────────┤
│  1. SQL 菜单 (system_menu 表)                           │
│     permission: system:user:query                       │
│     permission: system:user:create                      │
│     permission: system:user:update                      │
│     permission: system:user:delete                      │
│     permission: system:user:export                      │
│     permission: system:user:import                      │
├─────────────────────────────────────────────────────────┤
│  2. 后端 Controller (@PreAuthorize)                     │
│     @PreAuthorize("@ss.hasPermission('system:user:query')")  │
│     @PreAuthorize("@ss.hasPermission('system:user:create')") │
│     @PreAuthorize("@ss.hasPermission('system:user:update')") │
│     @PreAuthorize("@ss.hasPermission('system:user:delete')") │
│     @PreAuthorize("@ss.hasPermission('system:user:export')") │
│     @PreAuthorize("@ss.hasPermission('system:user:import')") │
├─────────────────────────────────────────────────────────┤
│  3. 前端按钮 (v-hasPermi)                               │
│     v-hasPermi="['system:user:create']"                 │
│     v-hasPermi="['system:user:delete']"                 │
│     v-hasPermi="['system:user:export']"                 │
│     v-hasPermi="['system:user:import']"                 │
└─────────────────────────────────────────────────────────┘
```

---

## 八、场景分析：status 字段（TINYINT + 字典）

### 8.1 场景构造

假设数据库中有以下表：

```sql
CREATE TABLE `infra_demo` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '编号',
  `name` varchar(64) NOT NULL COMMENT '名称',
  `status` tinyint NOT NULL COMMENT '状态',
  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  PRIMARY KEY (`id`)
) COMMENT='示例表';
```

在代码生成器中配置：
- `status` 字段的 `dictType` = `common_status`（常见的启用/禁用字典）
- 字典数据：`0=禁用, 1=启用`

### 8.2 生成结果分析

#### 8.2.1 字段类型映射过程

```
数据库: TINYINT
    ↓
MyBatis Plus 映射: Byte
    ↓
CodegenBuilder 特殊处理: Byte → Integer  (CodegenBuilder.java:133-136)
    ↓
最终 Java 类型: Integer
    ↓
TypeScript 类型: number  (所有数值类型统一)
```

#### 8.2.2 表单组件选择

```
字段名后缀: status
    ↓
COLUMN_HTML_TYPE_MAPPINGS 匹配: "status" → RADIO
    ↓
表单组件: radio (单选框)
```

#### 8.2.3 生成的代码片段

**DO 类**：
```java
/**
 * 状态
 * 枚举 {@link TODO common_status 对应的类}
 */
private Integer status;
```

**RespVO 类**（带字典注解）：
```java
@Schema(description = "状态", requiredMode = Schema.RequiredMode.REQUIRED, example = "1")
@ExcelProperty(value = "状态", converter = DictConvert.class)
@DictFormat("common_status")
private Integer status;
```

**Controller 权限**：
```java
@PreAuthorize("@ss.hasPermission('infra:demo:query')")
@PreAuthorize("@ss.hasPermission('infra:demo:create')")
@PreAuthorize("@ss.hasPermission('infra:demo:update')")
@PreAuthorize("@ss.hasPermission('infra:demo:delete')")
```

**TypeScript 接口**：
```typescript
export interface Demo {
  id?: number;        // 编号
  name: string;       // 名称
  status: number;     // 状态 (Integer → number)
  createTime?: string | Dayjs; // 创建时间
}
```

**前端查询表单**：
```vue
<el-form-item label="状态" prop="status">
  <el-select v-model="queryParams.status" placeholder="请选择状态" clearable>
    <el-option
      v-for="dict in getIntDictOptions(DICT_TYPE.COMMON_STATUS)"
      :key="dict.value"
      :label="dict.label"
      :value="dict.value"
    />
  </el-select>
</el-form-item>
```

**SQL 菜单**：
```sql
-- 菜单
INSERT INTO system_menu(name, permission, type, ...)
VALUES ('示例管理', '', 2, ...);

-- 按钮
INSERT INTO system_menu(name, permission, type, ...)
VALUES ('示例查询', 'infra:demo:query', 3, ...);
INSERT INTO system_menu(name, permission, type, ...)
VALUES ('示例创建', 'infra:demo:create', 3, ...);
INSERT INTO system_menu(name, permission, type, ...)
VALUES ('示例更新', 'infra:demo:update', 3, ...);
INSERT INTO system_menu(name, permission, type, ...)
VALUES ('示例删除', 'infra:demo:delete', 3, ...);
```

### 8.3 最容易出现不一致的地方

#### 8.3.1 类型不一致：TINYINT 与 Integer 的转换

**问题点**：
- 数据库：`TINYINT`（通常表示 0-127 的小整数）
- Java：`Integer`（经过 Byte → Integer 转换）
- TypeScript：`number`（所有数值类型统一）

**潜在风险**：
1. **字典值类型不匹配**：如果字典定义中 value 是 String 类型（如 "0", "1"），而 Java 是 Integer，前端可能出现类型比较错误
2. **前后端传值不一致**：后端期望 Integer，前端传的是 string 类型的数字
3. **枚举类定义错误**：如果手动创建枚举类，可能定义为 `int` 而不是 `Integer`

**验证方式**：
- 检查前端字典配置：`getIntDictOptions` vs `getStrDictOptions`
- 确认 `dictType` 对应的字典 value 类型是否与 Java 类型匹配

#### 8.3.2 字典类型配置不一致

**问题点**：`dictType` 字段需要手动配置，不是自动识别的

**潜在风险**：
1. **忘记配置 dictType**：表单会显示"请选择字典生成"的占位符，无法正常使用
2. **dictType 拼写错误**：如 `common_status` 写成 `common_state`，运行时报错
3. **字典不存在**：配置了 dictType 但数据库中没有对应的 `system_dict_type` 记录

**验证方式**：
- 检查 `CodegenColumnDO.dictType` 是否正确配置
- 确认 `system_dict_type` 表中存在对应记录
- 检查前端模板生成的 `DICT_TYPE.$dictType.toUpperCase()` 是否正确

#### 8.3.3 表单组件选择不一致

**问题点**：表单组件基于字段后缀自动匹配

**潜在风险**：
1. **字段名不匹配规则**：如果字段名为 `userStatus` 而非 `status`，需要确认是否以 `status` 结尾
2. **Java 类型覆盖规则**：如果 Java 类型是 `Boolean`，会被强制设置为 `radio`，忽略后缀规则
3. **手动修改后被覆盖**：如果手动修改了 htmlType，重新同步数据库时可能被覆盖

**验证方式**：
- 检查 `CodegenBuilder.processColumnUI()` 的执行逻辑
- 确认字段后缀是否在 `COLUMN_HTML_TYPE_MAPPINGS` 中
- 同步数据库后检查 htmlType 是否保持预期

#### 8.3.4 权限标识不一致

**问题点**：权限标识基于表名解析

**潜在风险**：
1. **className 与 moduleName 重复**：如表名 `system_system` 会产生特殊处理逻辑
2. **前后端权限格式不一致**：前端 `v-hasPermi` 与后端 `@PreAuthorize` 格式不同
3. **菜单 SQL 与代码不匹配**：手动修改了代码但忘记更新数据库菜单

**验证方式**：
- 检查 `CodegenEngine.initBindingMap()` 中的权限前缀计算
- 确认 Controller 中的 `@PreAuthorize` 注解值
- 检查前端 `v-hasPermi` 指令值
- 对比生成的 SQL 文件中的 permission 字段

#### 8.3.5 主子表/树表配置遗漏

**问题点**：特殊表类型需要额外配置字段

**潜在风险**：
- 树表：忘记配置 `treeParentColumnId` 和 `treeNameColumnId`
- 主子表：子表忘记配置 `masterTableId` 和 `subJoinColumnId`

**验证方式**：
- 检查 `CodegenTableDO` 的相关字段是否为 null
- 确认模板条件判断：`$table.templateType == 2`（树表）
- 验证生成的代码是否包含特殊逻辑（如 `children` 字段、ROOT 常量）

### 8.4 不一致问题检测清单

| 检查项 | 检查位置 | 预期值 | 实际值 | 状态 |
|--------|---------|--------|--------|------|
| Java 类型 | DO 类 | `Integer` | | ⬜ |
| TypeScript 类型 | API 接口 | `number` | | ⬜ |
| 表单组件 | 配置表 | `radio` | | ⬜ |
| 字典方法 | 前端代码 | `getIntDictOptions` | | ⬜ |
| dictType 配置 | 配置表 | `common_status` | | ⬜ |
| 字典类型存在 | system_dict_type | 有记录 | | ⬜ |
| 字典值类型 | system_dict_data | 数值型 | | ⬜ |
| 权限前缀 | Controller | `infra:demo` | | ⬜ |
| 菜单权限 | SQL 文件 | `infra:demo:*` | | ⬜ |
| 前端权限 | Vue 模板 | `infra:demo:*` | | ⬜ |

---

## 九、总结

### 9.1 代码生成器核心流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                        代码生成器全流程                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. 元数据读取阶段                                                    │
│     ┌──────────────┐    ┌─────────────────────┐                     │
│     │  MySQL 数据库 │───▶│ MyBatis Plus Generator │                  │
│     └──────────────┘    └─────────────────────┘                     │
│                                  │                                   │
│                                  ▼                                   │
│                         TableInfo / TableField                       │
│                                                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  2. 配置构建阶段                                                      │
│     ┌─────────────────────────────────────────────────┐              │
│     │              CodegenBuilder                      │              │
│     ├─────────────────────────────────────────────────┤              │
│     │  TableInfo ──▶ CodegenTableDO                   │              │
│     │    - moduleName = 表名前缀                       │              │
│     │    - businessName = 表名后缀                     │              │
│     │    - className = 驼峰类名                        │              │
│     │                                                 │              │
│     │  TableField ──▶ CodegenColumnDO                 │              │
│     │    - TINYINT ──▶ Byte ──▶ Integer (特殊处理)     │              │
│     │    - 字段后缀 ──▶ htmlType (表单组件)            │              │
│     │    - 字段后缀 ──▶ listOperationCondition        │              │
│     └─────────────────────────────────────────────────┘              │
│                                                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  3. 代码生成阶段                                                      │
│     ┌──────────────────────────────────────────────────────────┐    │
│     │                    CodegenEngine (Velocity)               │    │
│     ├──────────────────────────────────────────────────────────┤    │
│     │  模板组:                                                  │    │
│     │  ├── Java 模板 (后端 CRUD)                               │    │
│     │  │   ├── Controller (@PreAuthorize)                     │    │
│     │  │   ├── Service / Mapper                               │    │
│     │  │   ├── DO / VO / DTO                                  │    │
│     │  │   └── 单元测试                                        │    │
│     │  │                                                       │    │
│     │  ├── Vue 模板 (前端页面)                                 │    │
│     │  │   ├── index.vue (列表页)                              │    │
│     │  │   ├── form.vue (表单页)                               │    │
│     │  │   └── api.ts (API 接口)                               │    │
│     │  │                                                       │    │
│     │  └── SQL 模板 (菜单权限)                                 │    │
│     │      └── system_menu 插入语句                            │    │
│     └──────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 9.2 关键设计亮点

1. **模板驱动设计**：使用 Velocity 模板引擎，易于扩展和定制
2. **多前端框架支持**：支持 Vue 2、Vue 3、Vben、Uniapp 等多种前端框架
3. **多数据库支持**：MySQL、Oracle、PostgreSQL、SQL Server、达梦、人大金仓等
4. **特殊场景支持**：单表、树表、主子表（普通/ERP/内嵌三种模式）
5. **权限一体化**：前后端权限标识完全一致，通过 SQL 自动初始化菜单

### 9.3 常见问题与建议

| 问题类型 | 常见表现 | 排查建议 |
|---------|---------|---------|
| 类型映射错误 | 前后端传值类型不匹配 | 检查 CodegenBuilder 的 Byte→Integer 转换 |
| 字典不显示 | 前端下拉框为空 | 确认 dictType 配置且字典数据存在 |
| 权限不生效 | 403 无权限 | 对比 SQL 菜单、@PreAuthorize、v-hasPermi 三者 |
| 树表不显示层级 | 列表无树形结构 | 检查 treeParentColumnId 和 treeNameColumnId |
| 主子表关联失败 | 子表数据不显示 | 检查 masterTableId 和 subJoinColumnId |
| 表单组件错误 | 显示 input 而非 radio | 检查字段后缀是否匹配 COLUMN_HTML_TYPE_MAPPINGS |
