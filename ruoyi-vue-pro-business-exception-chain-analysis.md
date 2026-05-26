# RuoYi-Vue-Pro 业务异常完整链路分析

## 一、整体架构概览

### 1.1 异常处理核心流程

```
┌─────────────────────────────────────────────────────────────────┐
│                      前端 (Vue 前端)                              │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ 1. Vue 组件调用 API                                      │    │
│  └───────────────────────────┬─────────────────────────────┘    │
│                              │                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ 2. Request Interceptor                                   │    │
│  │    - 注入 Token/TenantId                                 │    │
│  │    - 请求去重/取消                                        │    │
│  └───────────────────────────┬─────────────────────────────┘    │
└──────────────────────────────┼──────────────────────────────────┘
                               │ HTTP Request
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                      后端 (Spring Boot)                           │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ 3. Controller 接收请求                                   │    │
│  │    - 参数校验 (@Valid)                                   │    │
│  │    - 权限校验 (@PreAuthorize)                            │    │
│  └───────────────────────────┬─────────────────────────────┘    │
│                              │                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ 4. Service 执行业务逻辑                                   │    │
│  │    - 业务判断失败 → throw ServiceException               │    │
│  └───────────────────────────┬─────────────────────────────┘    │
│                              │ Exception                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ 5. GlobalExceptionHandler 全局异常处理器                  │    │
│  │    - 捕获各种异常                                         │    │
│  │    - 包装成 CommonResult                                 │    │
│  └───────────────────────────┬─────────────────────────────┘    │
└──────────────────────────────┼──────────────────────────────────┘
                               │ HTTP 200 + CommonResult
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                      前端 (Vue 前端)                              │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ 6. Response Interceptor                                  │    │
│  │    - 检查 CommonResult.code                              │    │
│  │    - code = 0: 返回 data                                 │    │
│  │    - code != 0: 弹窗提示或跳转                            │    │
│  └───────────────────────────┬─────────────────────────────┘    │
│                              │                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ 7. Vue 组件处理结果                                       │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、ErrorCode 定义与模块化编号规则

### 2.1 ErrorCode 类定义

**位置：** `yudao-framework/yudao-common/src/main/java/cn/iocoder/yudao/framework/common/exception/ErrorCode.java`

```java
@Data
public class ErrorCode {
    /**
     * 错误码
     */
    private final Integer code;
    /**
     * 错误提示
     */
    private final String msg;

    public ErrorCode(Integer code, String message) {
        this.code = code;
        this.msg = message;
    }
}
```

### 2.2 错误码区间划分

| 区间 | 类型 | 说明 |
|------|------|------|
| **0** | 成功 | 业务成功，`GlobalErrorCodeConstants.SUCCESS` |
| **0-999** | 全局系统错误码 | HTTP 状态码映射 + 自定义错误码 |
| **1,000,000,000+** | 业务异常错误码 | 各业务模块自定义 |

### 2.3 全局错误码枚举

**位置：** `yudao-framework/yudao-common/src/main/java/cn/iocoder/yudao/framework/common/exception/enums/GlobalErrorCodeConstants.java`

```java
public interface GlobalErrorCodeConstants {
    // 成功
    ErrorCode SUCCESS = new ErrorCode(0, "成功");

    // ========== 客户端错误段 ==========
    ErrorCode BAD_REQUEST = new ErrorCode(400, "请求参数不正确");
    ErrorCode UNAUTHORIZED = new ErrorCode(401, "账号未登录");
    ErrorCode FORBIDDEN = new ErrorCode(403, "没有该操作权限");
    ErrorCode NOT_FOUND = new ErrorCode(404, "请求未找到");
    ErrorCode METHOD_NOT_ALLOWED = new ErrorCode(405, "请求方法不正确");
    ErrorCode LOCKED = new ErrorCode(423, "请求失败，请稍后重试");
    ErrorCode TOO_MANY_REQUESTS = new ErrorCode(429, "请求过于频繁，请稍后重试");

    // ========== 服务端错误段 ==========
    ErrorCode INTERNAL_SERVER_ERROR = new ErrorCode(500, "系统异常");
    ErrorCode NOT_IMPLEMENTED = new ErrorCode(501, "功能未实现/未开启");
    ErrorCode ERROR_CONFIGURATION = new ErrorCode(502, "错误的配置项");

    // ========== 自定义错误段 ==========
    ErrorCode REPEATED_REQUESTS = new ErrorCode(900, "重复请求，请稍后重试");
    ErrorCode DEMO_DENY = new ErrorCode(901, "演示模式，禁止写操作");
    ErrorCode UNKNOWN = new ErrorCode(999, "未知错误");
}
```

### 2.4 业务错误码模块化编号规则

**位置：** `yudao-framework/yudao-common/src/main/java/cn/iocoder/yudao/framework/common/exception/enums/ServiceErrorCodeRange.java`

业务错误码采用 **10 位数字分段设计**：

```
  1    002    003    000
  │      │      │      │
  │      │      │      └─── 第四段：错误码（3位），模块内自增
  │      │      └────────── 第三段：模块（3位），系统内的子模块
  │      └───────────────── 第二段：系统类型（3位），业务系统标识
  └──────────────────────── 第一段：类型（1位），1=业务级别异常
```

**各模块编号区间分配：**

| 模块 | 编号区间 | 示例 |
|------|---------|------|
| infra（基础设施） | 1-001-000-000 ~ 1-002-000-000 | 1_001_000_001 |
| system（系统管理） | 1-002-000-000 ~ 1-003-000-000 | 1_002_000_000 |
| report（报表） | 1-003-000-000 ~ 1-004-000-000 | 1_003_000_000 |
| member（会员） | 1-004-000-000 ~ 1-005-000-000 | 1_004_000_000 |
| product（商品） | 1-008-000-000 ~ 1-009-000-000 | 1_008_000_000 |
| pay（支付） | 1-007-000-000 ~ 1-008-000-000 | 1_007_000_000 |
| bpm（工作流） | 1-009-000-000 ~ 1-010-000-000 | 1_009_000_000 |
| trade（交易） | 1-011-000-000 ~ 1-012-000-000 | 1_011_000_000 |
| promotion（营销） | 1-013-000-000 ~ 1-014-000-000 | 1_013_000_000 |
| crm（客户关系） | 1-020-000-000 ~ 1-021-000-000 | 1_020_000_000 |
| ai（人工智能） | 1-022-000-000 ~ 1-023-000-000 | 1_022_000_000 |

### 2.5 业务模块错误码定义示例

**System 模块示例（位置：`yudao-module-system/src/main/java/cn/iocoder/yudao/module/system/enums/ErrorCodeConstants.java`）：**

```java
public interface ErrorCodeConstants {
    // ========== AUTH 模块 1-002-000-000 ==========
    ErrorCode AUTH_LOGIN_BAD_CREDENTIALS = new ErrorCode(1_002_000_000, "登录失败，账号密码不正确");
    ErrorCode AUTH_LOGIN_USER_DISABLED = new ErrorCode(1_002_000_001, "登录失败，账号被禁用");
    ErrorCode AUTH_LOGIN_CAPTCHA_CODE_ERROR = new ErrorCode(1_002_000_004, "验证码不正确，原因：{}");

    // ========== 菜单模块 1-002-001-000 ==========
    ErrorCode MENU_NAME_DUPLICATE = new ErrorCode(1_002_001_000, "已经存在该名字的菜单");
    ErrorCode MENU_NOT_EXISTS = new ErrorCode(1_002_001_003, "菜单不存在");

    // ========== 用户模块 1-002-003-000 ==========
    ErrorCode USER_USERNAME_EXISTS = new ErrorCode(1_002_003_000, "用户账号已经存在");
    ErrorCode USER_NOT_EXISTS = new ErrorCode(1_002_003_003, "用户不存在");
}
```

**Infra 模块示例（位置：`yudao-module-infra/src/main/java/cn/iocoder/yudao/module/infra/enums/ErrorCodeConstants.java`）：**

```java
public interface ErrorCodeConstants {
    // ========== 参数配置 1-001-000-000 ==========
    ErrorCode CONFIG_NOT_EXISTS = new ErrorCode(1_001_000_001, "参数配置不存在");
    ErrorCode CONFIG_KEY_DUPLICATE = new ErrorCode(1_001_000_002, "参数配置 key 重复");

    // ========== 定时任务 1-001-001-000 ==========
    ErrorCode JOB_NOT_EXISTS = new ErrorCode(1_001_001_000, "定时任务不存在");
    ErrorCode JOB_CRON_EXPRESSION_VALID = new ErrorCode(1_001_001_005, "CRON 表达式不正确");
}
```

---

## 三、ServiceException 构造方式

### 3.1 ServiceException 类定义

**位置：** `yudao-framework/yudao-common/src/main/java/cn/iocoder/yudao/framework/common/exception/ServiceException.java`

```java
@Data
@EqualsAndHashCode(callSuper = true)
public final class ServiceException extends RuntimeException {

    /**
     * 业务错误码
     */
    private Integer code;
    /**
     * 错误提示
     */
    private String message;

    /**
     * 空构造方法，避免反序列化问题
     */
    public ServiceException() {
    }

    /**
     * 通过 ErrorCode 构造
     */
    public ServiceException(ErrorCode errorCode) {
        this.code = errorCode.getCode();
        this.message = errorCode.getMsg();
    }

    /**
     * 通过 code 和 message 直接构造
     */
    public ServiceException(Integer code, String message) {
        this.code = code;
        this.message = message;
    }

    // ... setter/getter 方法
}
```

### 3.2 ServiceExceptionUtil 工具类

**位置：** `yudao-framework/yudao-common/src/main/java/cn/iocoder/yudao/framework/common/exception/util/ServiceExceptionUtil.java`

```java
@Slf4j
public class ServiceExceptionUtil {

    // ========== 和 ServiceException 的集成 ==========

    /**
     * 通过 ErrorCode 构造异常
     */
    public static ServiceException exception(ErrorCode errorCode) {
        return exception0(errorCode.getCode(), errorCode.getMsg());
    }

    /**
     * 通过 ErrorCode + 参数构造异常（支持占位符格式化）
     * 示例：errorCode.msg = "验证码不正确，原因：{}"
     *       params = ["图形验证码已过期"]
     *       结果："验证码不正确，原因：图形验证码已过期"
     */
    public static ServiceException exception(ErrorCode errorCode, Object... params) {
        return exception0(errorCode.getCode(), errorCode.getMsg(), params);
    }

    /**
     * 通过 code + messagePattern + 参数构造异常
     */
    public static ServiceException exception0(Integer code, String messagePattern, Object... params) {
        String message = doFormat(code, messagePattern, params);
        return new ServiceException(code, message);
    }

    /**
     * 快速构造参数校验异常（code = 400）
     */
    public static ServiceException invalidParamException(String messagePattern, Object... params) {
        return exception0(GlobalErrorCodeConstants.BAD_REQUEST.getCode(), messagePattern, params);
    }

    // ========== 格式化方法 ==========

    /**
     * 使用 {} 作为占位符进行格式化
     * 区别于 String.format，参数不正确时不会报错
     */
    @VisibleForTesting
    public static String doFormat(int code, String messagePattern, Object... params) {
        StringBuilder sbuf = new StringBuilder(messagePattern.length() + 50);
        int i = 0;
        int j;
        int l;
        for (l = 0; l < params.length; l++) {
            j = messagePattern.indexOf("{}", i);
            if (j == -1) {
                log.error("[doFormat][参数过多：错误码({})|错误内容({})|参数({})", code, messagePattern, params);
                if (i == 0) {
                    return messagePattern;
                } else {
                    sbuf.append(messagePattern.substring(i));
                    return sbuf.toString();
                }
            } else {
                sbuf.append(messagePattern, i, j);
                sbuf.append(params[l]);
                i = j + 2;
            }
        }
        if (messagePattern.indexOf("{}", i) != -1) {
            log.error("[doFormat][参数过少：错误码({})|错误内容({})|参数({})", code, messagePattern, params);
        }
        sbuf.append(messagePattern.substring(i));
        return sbuf.toString();
    }
}
```

### 3.3 Service 层抛出异常的常见方式

**方式一：使用 ErrorCode 枚举**

```java
@Service
public class UserServiceImpl implements UserService {

    @Override
    public void updateUser(Long id, UserUpdateReqVO reqVO) {
        // 校验用户是否存在
        UserDO user = userMapper.selectById(id);
        if (user == null) {
            throw new ServiceException(USER_NOT_EXISTS);
        }
        // ...
    }
}
```

**方式二：使用 ServiceExceptionUtil + 参数格式化**

```java
@Service
public class RoleServiceImpl implements RoleService {

    @Override
    public Long createRole(RoleCreateReqVO reqVO) {
        // 校验角色名是否唯一
        if (roleMapper.selectByName(reqVO.getName()) != null) {
            throw ServiceExceptionUtil.exception(ROLE_NAME_DUPLICATE, reqVO.getName());
        }
        // ...
    }
}
```

**方式三：使用 invalidParamException 快速构造参数异常**

```java
@Service
public class AuthServiceImpl implements AuthService {

    @Override
    public void login(String username, String password) {
        if (StrUtil.isBlank(username)) {
            throw ServiceExceptionUtil.invalidParamException("用户名不能为空");
        }
        // ...
    }
}
```

---

## 四、全局异常处理器包装 CommonResult

### 4.1 CommonResult 统一响应格式

**位置：** `yudao-framework/yudao-common/src/main/java/cn/iocoder/yudao/framework/common/pojo/CommonResult.java`

```java
@Data
public class CommonResult<T> implements Serializable {

    /**
     * 错误码
     */
    private Integer code;
    /**
     * 错误提示，用户可阅读
     */
    private String msg;
    /**
     * 返回数据
     */
    private T data;

    /**
     * 成功响应
     */
    public static <T> CommonResult<T> success(T data) {
        CommonResult<T> result = new CommonResult<>();
        result.code = GlobalErrorCodeConstants.SUCCESS.getCode();  // code = 0
        result.data = data;
        result.msg = "";
        return result;
    }

    /**
     * 错误响应 - 通过 code + message
     */
    public static <T> CommonResult<T> error(Integer code, String message) {
        Assert.notEquals(GlobalErrorCodeConstants.SUCCESS.getCode(), code, "code 必须是错误的！");
        CommonResult<T> result = new CommonResult<>();
        result.code = code;
        result.msg = message;
        return result;
    }

    /**
     * 错误响应 - 通过 ErrorCode
     */
    public static <T> CommonResult<T> error(ErrorCode errorCode) {
        return error(errorCode.getCode(), errorCode.getMsg());
    }

    /**
     * 错误响应 - 通过 ErrorCode + 参数格式化
     */
    public static <T> CommonResult<T> error(ErrorCode errorCode, Object... params) {
        CommonResult<T> result = new CommonResult<>();
        result.code = errorCode.getCode();
        result.msg = ServiceExceptionUtil.doFormat(errorCode.getCode(), errorCode.getMsg(), params);
        return result;
    }

    /**
     * 判断是否成功
     */
    public static boolean isSuccess(Integer code) {
        return Objects.equals(code, GlobalErrorCodeConstants.SUCCESS.getCode());
    }

    @JsonIgnore
    public boolean isSuccess() {
        return isSuccess(code);
    }

    @JsonIgnore
    public boolean isError() {
        return !isSuccess();
    }

    /**
     * 如果有错误则抛出 ServiceException
     */
    public void checkError() throws ServiceException {
        if (isSuccess()) {
            return;
        }
        throw new ServiceException(code, msg);
    }
}
```

**响应示例：**

```json
// 成功响应
{
  "code": 0,
  "msg": "",
  "data": {
    "id": 1,
    "username": "admin"
  }
}

// 业务错误响应（HTTP 状态码 = 200）
{
  "code": 1002003000,
  "msg": "用户账号已经存在",
  "data": null
}

// 参数校验错误响应（HTTP 状态码 = 200）
{
  "code": 400,
  "msg": "请求参数不正确:用户名不能为空",
  "data": null
}
```

### 4.2 GlobalExceptionHandler 全局异常处理器

**位置：** `yudao-framework/yudao-spring-boot-starter-web/src/main/java/cn/iocoder/yudao/framework/web/core/handler/GlobalExceptionHandler.java`

```java
@RestControllerAdvice
@AllArgsConstructor
@Slf4j
public class GlobalExceptionHandler {

    private final String applicationName;
    private final ApiErrorLogCommonApi apiErrorLogApi;

    /**
     * 处理所有异常，主要是提供给 Filter 使用
     * 因为 Filter 不走 SpringMVC 的流程
     */
    public CommonResult<?> allExceptionHandler(HttpServletRequest request, Throwable ex) {
        // 按异常类型分发处理
        if (ex instanceof MissingServletRequestParameterException) {
            return missingServletRequestParameterExceptionHandler((MissingServletRequestParameterException) ex);
        }
        if (ex instanceof MethodArgumentNotValidException) {
            return methodArgumentNotValidExceptionExceptionHandler((MethodArgumentNotValidException) ex);
        }
        if (ex instanceof ServiceException) {
            return serviceExceptionHandler((ServiceException) ex);
        }
        if (ex instanceof AccessDeniedException) {
            return accessDeniedExceptionHandler(request, (AccessDeniedException) ex);
        }
        return defaultExceptionHandler(request, ex);
    }

    /**
     * 处理业务异常 ServiceException
     * 例如说：商品库存不足，用户手机号已存在
     */
    @ExceptionHandler(value = ServiceException.class)
    public CommonResult<?> serviceExceptionHandler(ServiceException ex) {
        log.warn("[serviceExceptionHandler]\n\t{}", ex.getStackTrace()[0]);
        return CommonResult.error(ex.getCode(), ex.getMessage());
    }

    /**
     * 处理 Spring Security 权限不足的异常
     * 来源：使用 @PreAuthorize 注解，AOP 进行权限拦截
     */
    @ExceptionHandler(value = AccessDeniedException.class)
    public CommonResult<?> accessDeniedExceptionHandler(HttpServletRequest req, AccessDeniedException ex) {
        log.warn("[accessDeniedExceptionHandler][userId({}) 无法访问 url({})]", 
                WebFrameworkUtils.getLoginUserId(req), req.getRequestURL(), ex);
        return CommonResult.error(FORBIDDEN);  // code = 403
    }

    /**
     * 处理 SpringMVC 参数校验不正确（@Valid 触发）
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public CommonResult<?> methodArgumentNotValidExceptionExceptionHandler(MethodArgumentNotValidException ex) {
        log.warn("[methodArgumentNotValidExceptionExceptionHandler]", ex);
        // 获取错误信息
        String errorMessage = null;
        FieldError fieldError = ex.getBindingResult().getFieldError();
        if (fieldError == null) {
            List<ObjectError> allErrors = ex.getBindingResult().getAllErrors();
            if (CollUtil.isNotEmpty(allErrors)) {
                errorMessage = allErrors.get(0).getDefaultMessage();
            }
        } else {
            errorMessage = fieldError.getDefaultMessage();
        }
        // 转换 CommonResult
        if (StrUtil.isEmpty(errorMessage)) {
            return CommonResult.error(BAD_REQUEST);
        }
        return CommonResult.error(BAD_REQUEST.getCode(), String.format("请求参数不正确:%s", errorMessage));
    }

    /**
     * 处理 Validator 校验不通过产生的异常
     */
    @ExceptionHandler(value = ConstraintViolationException.class)
    public CommonResult<?> constraintViolationExceptionHandler(ConstraintViolationException ex) {
        log.warn("[constraintViolationExceptionHandler]", ex);
        ConstraintViolation<?> constraintViolation = ex.getConstraintViolations().iterator().next();
        return CommonResult.error(BAD_REQUEST.getCode(), String.format("请求参数不正确:%s", constraintViolation.getMessage()));
    }

    /**
     * 处理 SpringMVC 请求参数缺失
     */
    @ExceptionHandler(value = MissingServletRequestParameterException.class)
    public CommonResult<?> missingServletRequestParameterExceptionHandler(MissingServletRequestParameterException ex) {
        log.warn("[missingServletRequestParameterExceptionHandler]", ex);
        return CommonResult.error(BAD_REQUEST.getCode(), String.format("请求参数缺失:%s", ex.getParameterName()));
    }

    /**
     * 处理系统异常，兜底处理所有的一切
     */
    @ExceptionHandler(value = Exception.class)
    public CommonResult<?> defaultExceptionHandler(HttpServletRequest req, Throwable ex) {
        // 特殊：如果是 ServiceException 的包装异常
        if (ex.getCause() != null && ex.getCause() instanceof ServiceException) {
            return serviceExceptionHandler((ServiceException) ex.getCause());
        }

        // 处理表不存在的异常（模块未开启）
        CommonResult<?> tableNotExistsResult = handleTableNotExists(ex);
        if (tableNotExistsResult != null) {
            return tableNotExistsResult;
        }

        // 记录异常日志
        log.error("[defaultExceptionHandler]", ex);
        createExceptionLog(req, ex);

        // 返回系统异常
        return CommonResult.error(INTERNAL_SERVER_ERROR.getCode(), INTERNAL_SERVER_ERROR.getMsg());
    }

    /**
     * 处理模块表不存在的情况（提示用户开启模块）
     */
    private CommonResult<?> handleTableNotExists(Throwable ex) {
        String message = ExceptionUtil.getRootCauseMessage(ex);
        if (!message.contains("doesn't exist")) {
            return null;
        }
        if (message.contains("bpm_")) {
            return CommonResult.error(NOT_IMPLEMENTED.getCode(),
                    "[工作流模块 yudao-module-bpm - 表结构未导入][参考 https://cloud.iocoder.cn/bpm/ 开启]");
        }
        if (message.contains("pay_")) {
            return CommonResult.error(NOT_IMPLEMENTED.getCode(),
                    "[支付模块 yudao-module-pay - 表结构未导入][参考 https://cloud.iocoder.cn/pay/build/ 开启]");
        }
        // ... 其他模块类似
        return null;
    }
}
```

---

## 五、参数校验异常 vs 权限异常：如何区分

### 5.1 参数校验异常（Bad Request）

**触发场景：**
- `@Valid` / `@Validated` 注解触发的 Bean Validation
- `@RequestParam` 参数缺失
- 参数类型不匹配
- 自定义的参数校验逻辑

**错误码：** `400`（`GlobalErrorCodeConstants.BAD_REQUEST`）

**代码示例：**

```java
// Controller 层 - 使用 @Valid 触发参数校验
@PostMapping("/create")
public CommonResult<Long> createUser(@Valid @RequestBody UserCreateReqVO reqVO) {
    Long id = userService.createUser(reqVO);
    return CommonResult.success(id);
}

// VO 类 - 使用 Validation 注解
@Data
public class UserCreateReqVO {
    @NotBlank(message = "用户名不能为空")
    @Length(max = 30, message = "用户名长度不能超过 30 个字符")
    private String username;

    @NotBlank(message = "密码不能为空")
    @Length(min = 4, max = 16, message = "密码长度为 4-16 位")
    private String password;

    @Email(message = "邮箱格式不正确")
    private String email;
}
```

**响应示例：**
```json
{
  "code": 400,
  "msg": "请求参数不正确:用户名不能为空",
  "data": null
}
```

### 5.2 权限异常（Forbidden）

**触发场景：**
- Spring Security 的 `@PreAuthorize` 注解
- 自定义的权限校验逻辑
- `AccessDeniedHandlerImpl` 处理 Filter 级别的权限拒绝

**错误码：** `403`（`GlobalErrorCodeConstants.FORBIDDEN`）

**代码示例：**

```java
// Controller 层 - 使用 @PreAuthorize 进行权限校验
@PreAuthorize("@ss.hasPermission('system:user:create')")
@PostMapping("/create")
public CommonResult<Long> createUser(@Valid @RequestBody UserCreateReqVO reqVO) {
    Long id = userService.createUser(reqVO);
    return CommonResult.success(id);
}
```

**响应示例：**
```json
{
  "code": 403,
  "msg": "没有该操作权限",
  "data": null
}
```

### 5.3 区分方式汇总

| 维度 | 参数校验异常 | 权限异常 |
|------|------------|---------|
| **错误码** | `400` | `403` |
| **触发时机** | Controller 参数绑定时 | 权限拦截器/AOP 校验时 |
| **异常类型** | `MethodArgumentNotValidException` / `ConstraintViolationException` | `AccessDeniedException` |
| **处理器方法** | `methodArgumentNotValidExceptionExceptionHandler()` | `accessDeniedExceptionHandler()` |
| **错误信息前缀** | "请求参数不正确:" / "请求参数缺失:" | "没有该操作权限" |
| **前端处理** | 弹窗提示用户检查输入 | 弹窗提示或跳转无权限页面 |

---

## 六、前端如何根据 code/message 弹窗或跳转

### 6.1 前端请求拦截器架构

RuoYi-Vue-Pro 使用 Axios 的拦截器机制处理响应：

```
                    Axios Response Interceptor
                              │
              ┌───────────────┼───────────────┐
              │               │               │
        HTTP 200          HTTP 4xx         HTTP 5xx
              │               │               │
              ▼               ▼               ▼
    ┌─────────────────┐ ┌─────────────┐ ┌───────────────┐
    │ 检查 code       │ │ 401: 跳转   │ │ 弹窗: 服务    │
    │ code = 0?       │ │      登录   │ │       器错误  │
    │                 │ │ 403: 弹窗   │ │               │
    └────────┬────────┘ └─────────────┘ └───────────────┘
             │
        ┌────┴────┐
        │         │
      Yes         No
        │         │
        ▼         ▼
   返回 data    弹窗提示
               (ElMessage.error)
```

### 6.2 Response Interceptor 核心逻辑

```typescript
// @/config/axios.ts 中的响应拦截器

axiosInstance.interceptors.response.use(
  (response: AxiosResponse) => {
    const res = response.data  // res = { code, msg, data }
    const config = response.config as any

    // 1. 如果配置了 isReturnOriginal，直接返回原始响应
    if (config.isReturnOriginal) {
      return response
    }

    // 2. 如果是下载请求（blob），直接返回
    if (config.responseType === 'blob') {
      return response
    }

    // 3. 检查业务状态码
    if (res.code === 0) {
      // 成功：返回 data 字段（自动解包）
      return res.data
    }

    // 4. 业务错误处理（HTTP 200 但 code != 0）
    return handleBusinessError(res, config)
  },
  (error: AxiosError) => {
    // HTTP 状态码非 2xx 时进入这里
    return handleHttpError(error)
  }
)
```

### 6.3 业务错误处理（HTTP 200 + code != 0）

```typescript
const handleBusinessError = (res: any, config: any) => {
  const { code, msg } = res

  // 1. 检查是否需要隐藏错误提示
  const isHideErrorMessage = config.isHideErrorMessage === true

  if (!isHideErrorMessage) {
    // 显示错误弹窗
    ElMessage.error(msg || '请求失败')
  }

  // 2. 特殊错误码处理
  switch (code) {
    case 401:
      // Token 过期，跳转登录
      return handle401Error(config)
    case 403:
      // 无权限
      return Promise.reject(new Error(msg || '无权限访问'))
    default:
      return Promise.reject(new Error(msg || '请求失败'))
  }
}
```

### 6.4 HTTP 错误处理（HTTP 4xx/5xx）

```typescript
const handleHttpError = (error: AxiosError) => {
  const config = error.config as any
  const response = error.response

  // 1. 请求被取消（AbortController.abort）
  if (axios.isCancel(error)) {
    console.log('Request canceled:', error.message)
    return Promise.reject(error)
  }

  // 2. 网络错误（无网络连接）
  if (!response) {
    ElMessage.error('网络连接失败，请检查网络')
    return Promise.reject(error)
  }

  const status = response.status
  const data = response.data as any

  // 3. 根据 HTTP 状态码处理
  switch (status) {
    case 401:
      return handle401Error(config)
    case 403:
      ElMessage.error('无权限访问')
      return Promise.reject(error)
    case 404:
      ElMessage.error('请求资源不存在')
      return Promise.reject(error)
    case 500:
      ElMessage.error(data?.msg || '服务器内部错误')
      return Promise.reject(error)
    default:
      ElMessage.error(data?.msg || `请求失败 (${status})`)
      return Promise.reject(error)
  }
}
```

### 6.5 401 处理与 Token 刷新

```typescript
// 标记是否正在刷新 Token
let isRefreshing = false
// 等待刷新完成的请求队列
const pendingRequests: Array<(token: string) => void> = []

const handle401Error = async (config: any) => {
  const refreshToken = getRefreshToken()

  // 1. 如果没有 refreshToken，直接跳转登录
  if (!refreshToken) {
    redirectToLogin()
    return Promise.reject(new Error('登录已过期'))
  }

  // 2. 如果不是正在刷新，开始刷新流程
  if (!isRefreshing) {
    isRefreshing = true

    refreshTokenRequest(refreshToken)
      .then((newToken) => {
        // 刷新成功，执行所有等待中的请求
        isRefreshing = false
        pendingRequests.forEach((callback) => callback(newToken))
        pendingRequests.length = 0
        return newToken
      })
      .catch(() => {
        // 刷新失败，跳转登录
        isRefreshing = false
        pendingRequests.length = 0
        redirectToLogin()
        return Promise.reject(new Error('Token 刷新失败'))
      })
  }

  // 3. 返回一个 Promise，等待刷新完成后重试
  return new Promise((resolve) => {
    pendingRequests.push((newToken: string) => {
      config.headers['Authorization'] = `Bearer ${newToken}`
      resolve(axiosInstance(config))
    })
  })
}

const redirectToLogin = () => {
  // 清除 Token
  localStorage.removeItem('access_token')
  localStorage.removeItem('refresh_token')

  // 清除 Store 状态
  const userStore = useUserStore()
  userStore.resetUser()

  // 跳转登录页，携带重定向地址
  const currentPath = router.currentRoute.value.fullPath
  router.push({
    path: '/login',
    query: {
      redirect: currentPath !== '/login' ? currentPath : '/'
    }
  })
}
```

### 6.6 业务层调用示例

```typescript
// API 定义
export const UserApi = {
  createUser: async (data: UserCreateReqVO) => {
    return await request.post({ url: '/system/user/create', data })
  }
}

// Vue 组件中使用
<script setup lang="ts">
const handleCreate = async () => {
  try {
    // 1. 调用 API
    const userId = await UserApi.createUser(formData.value)
    
    // 2. 成功处理（只有 code = 0 才会走到这里）
    ElMessage.success('创建成功')
    router.push({ path: '/system/user/index' })
  } catch (error) {
    // 3. 错误处理（通常不需要，因为拦截器已经弹窗了）
    // 只有配置了 isHideErrorMessage: true 时才需要自己处理
    console.error('创建失败:', error)
  }
}
</script>
```

---

## 七、错误场景：HTTP 200 但 CommonResult.code 非 0

### 7.1 场景构造

**后端代码：**

```java
@Service
public class UserServiceImpl implements UserService {

    @Override
    public Long createUser(UserCreateReqVO reqVO) {
        // 1. 校验用户名是否已存在
        if (userMapper.selectByUsername(reqVO.getUsername()) != null) {
            // 抛出业务异常
            throw new ServiceException(USER_USERNAME_EXISTS);
            // USER_USERNAME_EXISTS = 1_002_003_000, "用户账号已经存在"
        }

        // 2. 其他业务逻辑...
        return user.getId();
    }
}

@RestController
@RequestMapping("/system/user")
public class UserController {

    @PostMapping("/create")
    public CommonResult<Long> createUser(@Valid @RequestBody UserCreateReqVO reqVO) {
        Long id = userService.createUser(reqVO);
        return CommonResult.success(id);
    }
}
```

**全局异常处理器捕获：**

```java
@ExceptionHandler(value = ServiceException.class)
public CommonResult<?> serviceExceptionHandler(ServiceException ex) {
    return CommonResult.error(ex.getCode(), ex.getMessage());
    // 返回 CommonResult(code=1002003000, msg="用户账号已经存在", data=null)
}
```

**HTTP 响应：**

```
HTTP/1.1 200 OK
Content-Type: application/json
Date: Sun, 10 May 2026 00:00:00 GMT

{
  "code": 1002003000,
  "msg": "用户账号已经存在",
  "data": null
}
```

### 7.2 如果前端只看 HTTP 状态码会怎样？

**错误的前端代码（只检查 HTTP 状态码）：**

```typescript
// ❌ 错误示例：只检查 HTTP 状态码
const createUserWrong = async (data) => {
  try {
    const response = await axios.post('/system/user/create', data)
    
    // 只检查 HTTP 状态码，忽略业务状态码
    if (response.status === 200) {
      // ❌ 误判为成功！
      console.log('创建成功！', response.data)
      ElMessage.success('创建成功')  // 错误的弹窗！
      return response.data
    }
  } catch (error) {
    // ❌ 这里永远不会执行，因为 HTTP 是 200
    console.error('创建失败:', error)
    ElMessage.error('创建失败')
  }
}
```

### 7.3 会出现的误判问题

| 问题类型 | 具体表现 | 影响 |
|---------|---------|------|
| **误判成功** | HTTP 200 就认为成功，忽略 `code = 1002003000` | 用户看到"创建成功"弹窗，但实际数据未保存 |
| **错误数据处理** | 使用了 `response.data`，但 `data` 是 `null` | 前端可能因访问 `data.id` 导致报错 |
| **状态不一致** | 前端显示成功，后端实际失败 | 用户体验差，数据丢失 |
| **错误流程继续** | 继续执行成功后的逻辑（如跳转、刷新列表） | 页面跳转后发现数据不存在，造成困惑 |

### 7.4 正确的处理方式

**方式一：使用 RuoYi-Vue-Pro 封装的 request（推荐）**

```typescript
// ✅ 正确示例：使用项目封装的 request，自动检查 code
import request from '@/config/axios'

const createUserCorrect = async (data) => {
  try {
    // request 内部已在拦截器中检查了 code
    // 只有 code = 0 才会返回 data，否则会 reject
    const userId = await request.post({ 
      url: '/system/user/create', 
      data 
    })
    
    // 这里只有真正成功才会执行
    console.log('创建成功，用户ID:', userId)
    ElMessage.success('创建成功')
    return userId
  } catch (error) {
    // 拦截器已经弹窗了，这里可以做额外的清理工作
    console.error('创建失败:', error)
  }
}
```

**方式二：手动检查 CommonResult.code**

```typescript
// ✅ 正确示例：手动检查业务状态码
import axios from 'axios'

const createUserManual = async (data) => {
  try {
    const response = await axios.post('/system/user/create', data)
    const result = response.data  // { code, msg, data }

    // ✅ 必须检查业务状态码 code
    if (result.code === 0) {
      console.log('创建成功，用户ID:', result.data)
      ElMessage.success('创建成功')
      return result.data
    } else {
      // ❌ HTTP 200 但业务失败
      console.error('创建失败:', result.msg)
      ElMessage.error(result.msg)
      throw new Error(result.msg)
    }
  } catch (error) {
    console.error('请求失败:', error)
    throw error
  }
}
```

### 7.5 为什么后端返回 HTTP 200 而不是 4xx/5xx？

**设计理念：**

1. **HTTP 状态码 vs 业务状态码分离**
   - HTTP 状态码：表示传输层是否成功
   - 业务状态码：表示业务逻辑是否成功

2. **统一错误处理**
   - 所有错误都包装成 `CommonResult`，格式一致
   - 前端只需要处理一种响应格式

3. **RESTful vs 实际业务**
   - 严格的 RESTful 会用 HTTP 400/403/500 等
   - 但实际项目中，业务错误类型远多于 HTTP 状态码
   - 模块化错误码（10位）能提供更精确的错误信息

**对比：**

```json
// 传统 RESTful 方式（HTTP 400）
HTTP/1.1 400 Bad Request
{
  "timestamp": "2026-05-10T00:00:00",
  "status": 400,
  "error": "Bad Request",
  "message": "用户账号已经存在",
  "path": "/system/user/create"
}

// RuoYi-Vue-Pro 方式（HTTP 200 + 业务错误码）
HTTP/1.1 200 OK
{
  "code": 1002003000,
  "msg": "用户账号已经存在",
  "data": null
}
```

**优缺点：**

| 维度 | HTTP 状态码方式 | 业务状态码方式（RuoYi-Vue-Pro） |
|------|----------------|------------------------------|
| **语义清晰度** | 利用 HTTP 标准语义 | 需要自定义语义 |
| **错误码数量** | 有限（几十个） | 无限（10位数字） |
| **模块化管理** | 困难 | 容易（分段设计） |
| **前端处理** | 需要同时处理 HTTP 和业务 | 统一处理 CommonResult |
| **缓存行为** | 4xx/5xx 可能影响缓存 | HTTP 200 正常缓存逻辑 |

---

## 八、完整链路代码追踪

### 8.1 业务异常抛出链路

```
UserController.createUser()
    │
    ▼
UserServiceImpl.createUser()
    │
    ├── 检查用户名是否存在
    │       │
    │       ▼
    │   userMapper.selectByUsername()
    │       │
    │       └── 返回已存在的用户
    │
    └── throw ServiceException(USER_USERNAME_EXISTS)
            │ code = 1_002_003_000
            │ msg = "用户账号已经存在"
            │
            ▼
GlobalExceptionHandler.serviceExceptionHandler()
            │
            └── return CommonResult.error(code, msg)
                    │
                    ▼
            HTTP 200 + JSON:
            {
              "code": 1002003000,
              "msg": "用户账号已经存在",
              "data": null
            }
```

### 8.2 关键文件位置汇总

| 组件 | 文件路径 | 说明 |
|------|---------|------|
| ErrorCode | `yudao-framework/yudao-common/.../exception/ErrorCode.java` | 错误码对象 |
| GlobalErrorCodeConstants | `yudao-framework/yudao-common/.../exception/enums/GlobalErrorCodeConstants.java` | 全局错误码枚举 |
| ServiceErrorCodeRange | `yudao-framework/yudao-common/.../exception/enums/ServiceErrorCodeRange.java` | 业务错误码区间定义 |
| ServiceException | `yudao-framework/yudao-common/.../exception/ServiceException.java` | 业务异常类 |
| ServiceExceptionUtil | `yudao-framework/yudao-common/.../exception/util/ServiceExceptionUtil.java` | 异常构造工具类 |
| CommonResult | `yudao-framework/yudao-common/.../pojo/CommonResult.java` | 统一响应格式 |
| GlobalExceptionHandler | `yudao-framework/yudao-spring-boot-starter-web/.../handler/GlobalExceptionHandler.java` | 全局异常处理器 |
| AccessDeniedHandlerImpl | `yudao-framework/yudao-spring-boot-starter-security/.../handler/AccessDeniedHandlerImpl.java` | 权限拒绝处理器 |
| 模块 ErrorCodeConstants | `yudao-module-xxx/.../enums/ErrorCodeConstants.java` | 各模块错误码定义 |
| 前端 Request 封装 | `src/config/axios.ts` 或 `src/utils/request.ts` | Axios 拦截器 |

---

## 九、总结

### 9.1 核心要点

1. **错误码设计**：采用 10 位分段设计，支持模块化管理
2. **异常构造**：通过 `ServiceException` + `ServiceExceptionUtil` 支持参数格式化
3. **统一响应**：所有响应都包装成 `CommonResult`，格式为 `{ code, msg, data }`
4. **全局处理**：`GlobalExceptionHandler` 捕获所有异常，统一转换为 `CommonResult`
5. **HTTP 200**：业务异常不改变 HTTP 状态码，始终返回 200，通过 `code` 区分
6. **前端拦截**：Response Interceptor 检查 `code`，`code = 0` 才认为成功

### 9.2 容易踩坑的点

| 坑点 | 说明 | 正确做法 |
|------|------|---------|
| **只看 HTTP 状态码** | HTTP 200 不代表业务成功 | 必须检查 `CommonResult.code` |
| **错误码重复** | 各模块可能定义相同的 code | 严格遵守编号区间规则 |
| **异常被吞** | try-catch 后忘记抛或记录 | 业务异常直接抛出，不要 catch |
| **前端错误弹窗不显示** | 配置了 `isHideErrorMessage` | 检查 API 调用配置 |
| **401 并发问题** | 多个请求同时 401，多次刷新 Token | 使用 Promise 队列 + 锁 |

### 9.3 最佳实践

**后端：**
```java
// ✅ 推荐：使用 ErrorCode 枚举 + ServiceExceptionUtil
throw ServiceExceptionUtil.exception(USER_NOT_EXISTS, userId);

// ✅ 推荐：参数校验异常使用专用方法
throw ServiceExceptionUtil.invalidParamException("用户名不能为空");

// ❌ 不推荐：直接抛出 RuntimeException
throw new RuntimeException("用户不存在");
```

**前端：**
```typescript
// ✅ 推荐：使用项目封装的 request
const data = await request.post({ url, data })

// ✅ 推荐：需要自定义错误处理时
await request.post({ 
  url, 
  data, 
  isHideErrorMessage: true  // 自己处理错误
})

// ❌ 不推荐：直接使用原生 axios
await axios.post(url, data)  // 没有 code 检查
```

---

**文档生成时间**: 2026-05-10  
**分析基础**: ruoyi-vue-pro 当前代码库版本
