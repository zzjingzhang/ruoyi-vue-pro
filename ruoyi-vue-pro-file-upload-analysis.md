# Ruoyi-Vue-Pro 文件上传模块深度分析

## 一、文件上传完整流程

### 1.1 整体架构图

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   前端      │ ──▶ │ FileController │ ──▶ │  FileService  │ ──▶ │  FileClient  │
│ 选择文件     │     │  接收请求      │     │  业务逻辑      │     │  实际存储      │
└─────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
                         │                      │                      │
                         ▼                      ▼                      ▼
                    校验文件              生成唯一路径           上传到存储
                    大小/目录             保存FileDO             返回URL
```

### 1.2 两种上传模式

#### 模式一：后端中转上传（传统模式）

```
前端 ──MultipartFile──▶ FileController.uploadFile()
                           │
                           ▼
                    读取文件内容 byte[]
                           │
                           ▼
                    FileService.createFile()
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         生成路径     上传到存储      保存FileDO
         (日期/目录)  (FileClient)   (fileMapper)
              │            │            │
              └────────────┼────────────┘
                           ▼
                      返回 URL
```

**关键代码位置**：
- 控制器：`yudao-module-infra/src/main/java/cn/iocoder/yudao/module/infra/controller/admin/file/FileController.java:46-55`
- 服务层：`yudao-module-infra/src/main/java/cn/iocoder/yudao/module/infra/service/file/FileServiceImpl.java:70-101`

#### 模式二：前端直传云存储（预签名模式）

```
前端 ──GET /presigned-url──▶ 后端
                                   │
                                   ▼
                         FileService.presignPutUrl()
                                   │
                                   ▼
                         S3FileClient 生成预签名URL
                                   │
                                   ▼
                         返回 {uploadUrl, url, path}
                                   │
前端 ──PUT uploadUrl──▶ 云存储（OSS/MinIO等）
           │
           ▼
      上传成功
           │
           ▼
前端 ──POST /create──▶ 后端记录 FileDO
```

**关键代码位置**：
- 预签名：`FileController.java:57-67`
- 创建记录：`FileController.java:69-73`
- 实现逻辑：`FileServiceImpl.java:140-166`

---

## 二、文件大小和类型校验

### 2.1 文件大小校验

**全局配置位置**：`yudao-server/src/main/resources/application.yaml:14-16`

```yaml
spring:
  servlet:
    multipart:
      max-file-size: 16MB      # 单个文件最大 16MB
      max-request-size: 32MB   # 单次请求总大小 32MB
```

**校验机制**：
- 由 Spring MVC 的 `StandardServletMultipartResolver` 在进入 Controller 前执行
- 超过限制会抛出 `MaxUploadSizeExceededException`
- 全局异常处理器捕获并返回友好错误

### 2.2 文件目录安全校验

**位置**：`FileUploadReqVO.java:23-34`

```java
@AssertTrue(message = "文件目录不正确")
public boolean isDirectoryValid() {
    return !StrUtil.contains(directory, "..")    // 防止目录穿越
            && !StrUtil.startWithAny(directory, "/", "\\");  // 防止上传到根目录
}
```

**安全措施**：
- 禁止目录穿越攻击（`../`）
- 禁止绝对路径（以 `/` 或 `\` 开头）
- 确保文件只能上传到预设的基础路径内

### 2.3 文件类型识别

**位置**：`FileTypeUtils.java:20-109`

使用 Apache Tika 库进行文件类型检测：

```java
public static String getMineType(byte[] data, String name) {
    return TIKA.detect(data, name);  // 结合内容和文件名双重检测
}
```

**特点**：
- 基于文件魔数（Magic Number）检测真实类型
- 防止文件扩展名伪造（如把 `.exe` 改成 `.jpg`）
- 自动推断 MIME 类型和文件扩展名

---

## 三、存储配置选择机制

### 3.1 多存储配置架构

```
infra_file_config 表
├── id: 配置编号
├── name: 配置名称
├── storage: 存储器类型（枚举）
├── master: 是否为主配置
└── config: 具体配置（JSON格式）
```

### 3.2 支持的存储类型

| 类型 | 枚举值 | 配置类 | 适用场景 |
|-----|-------|-------|---------|
| 本地存储 | LOCAL | LocalFileClientConfig | 开发环境、小文件 |
| 数据库 | DB | DBFileClientConfig | 简单场景、无需额外服务 |
| FTP | FTP | FtpFileClientConfig | 传统文件服务器 |
| SFTP | SFTP | SftpFileClientConfig | 安全文件传输 |
| S3 协议 | S3 | S3FileClientConfig | MinIO、阿里云OSS、腾讯云COS、AWS S3 |

**核心接口**：`FileClient.java`

```java
public interface FileClient {
    Long getId();
    String upload(byte[] content, String path, String type);
    void delete(String path);
    byte[] getContent(String path);
    default String presignPutUrl(String path) { ... }
    default String presignGetUrl(String url, Integer expirationSeconds) { ... }
}
```

### 3.3 主配置机制

```java
// FileServiceImpl.java:92
FileClient client = fileConfigService.getMasterFileClient();
```

- 系统支持配置多个存储配置
- 只有一个被标记为 `master=true`
- 上传操作默认使用主配置
- 每个文件记录 `configId` 关联其使用的配置

---

## 四、本地存储 vs 云存储差异

### 4.1 本地存储实现

**位置**：`LocalFileClient.java:14-55`

```java
public String upload(byte[] content, String path, String type) {
    String filePath = getFilePath(path);
    FileUtil.writeBytes(content, filePath);
    return super.formatFileUrl(config.getDomain(), path);
}

private String getFilePath(String path) {
    return config.getBasePath() + File.separator + path;
}
```

**URL 格式**：
```
{domain}/admin-api/infra/file/{configId}/get/{path}
```

**示例**：
```
http://localhost:48080/admin-api/infra/file/1/get/20260510/test.jpg
```

**访问机制**：
- 通过 `FileController.getFileContent()` 接口代理访问
- 接口路径：`GET /infra/file/{configId}/get/**`
- 控制器标记 `@PermitAll`，无需登录即可访问

### 4.2 S3 云存储实现

**位置**：`S3FileClient.java:32-250`

#### 公开访问模式（enablePublicAccess=true）

```java
public String presignGetUrl(String url, Integer expirationSeconds) {
    // 公开访问：直接返回静态 URL
    return config.getDomain() + "/" + path;
}
```

**URL 格式**：
```
https://{bucket}.oss-cn-beijing.aliyuncs.com/20260510/test.jpg
```

#### 私有访问模式（enablePublicAccess=false）

```java
public String presignGetUrl(String url, Integer expirationSeconds) {
    // 私有访问：生成预签名 URL，默认 24 小时有效
    Duration expiration = expirationSeconds != null 
        ? Duration.ofSeconds(expirationSeconds) 
        : EXPIRATION_DEFAULT;  // 24 小时
    
    return presigner.presignGetObject(...)
        .signatureDuration(expiration)
        .build().url().toString();
}
```

**URL 格式**（带签名参数）：
```
https://bucket.oss-cn-beijing.aliyuncs.com/20260510/test.jpg
?X-Amz-Algorithm=AWS4-HMAC-SHA256
&X-Amz-Credential=...
&X-Amz-Date=...
&X-Amz-Expires=86400
&X-Amz-Signature=...
```

### 4.3 对比总结

| 特性 | 本地存储 | S3 公开 | S3 私有 |
|-----|---------|---------|---------|
| 文件位置 | 服务器磁盘 | 云服务商 | 云服务商 |
| URL 类型 | 代理接口 | 直接访问 | 预签名 |
| URL 有效期 | 永久（只要文件存在） | 永久（只要文件存在） | 默认 24 小时 |
| 访问控制 | 无（接口 PermitAll） | 无（公开读取） | 签名验证 |
| 前端直传 | 不支持 | 支持 | 支持 |
| 带宽消耗 | 占用服务器带宽 | 不占用 | 不占用 |

---

## 五、URL 长期可访问性分析

### 5.1 影响 URL 可访问性的因素

#### 因素一：存储配置变更

```
FileDO.url = 上传时生成的完整 URL
         ↓
如果后续切换存储配置，旧 URL 仍然指向旧存储
         ↓
只要旧存储配置存在且文件未删除，URL 仍然可用
```

**关键点**：
- `FileDO.configId` 记录使用的配置
- 删除配置但不删除物理文件 → URL 可能失效
- 修改配置的 domain/basePath → URL 失效

#### 因素二：文件物理删除

通过 `FileService.deleteFile()` 删除时：

```java
// FileServiceImpl.java:174-185
public void deleteFile(Long id) throws Exception {
    FileDO file = validateFileExists(id);
    
    // 1. 从存储删除物理文件
    FileClient client = fileConfigService.getFileClient(file.getConfigId());
    client.delete(file.getPath());
    
    // 2. 删除数据库记录
    fileMapper.deleteById(id);
}
```

**结论**：正常删除流程会同时删除物理文件和数据库记录。

### 5.2 各存储类型的 URL 有效期

| 存储类型 | 访问模式 | URL 类型 | 有效期 | 过期原因 |
|---------|---------|---------|--------|---------|
| 本地/DB/FTP | - | 代理 URL | 永久（文件存在时） | 文件被删除、配置变更 |
| S3 | 公开 | 静态 URL | 永久（文件存在时） | 文件被删除、Bucket 策略变更 |
| S3 | 私有 | 预签名 URL | 默认 24 小时 | 签名过期、文件被删除 |

---

## 六、私有桶签名 URL 过期风险

### 6.1 签名过期机制

```java
// S3FileClient.java:34
private static final Duration EXPIRATION_DEFAULT = Duration.ofHours(24);

// S3FileClient.java:130
Duration expiration = expirationSeconds != null 
    ? Duration.ofSeconds(expirationSeconds) 
    : EXPIRATION_DEFAULT;
```

### 6.2 过期风险场景

#### 场景 A：前端缓存过期 URL

```
1. 用户上传私密文档，获得预签名 URL（有效期 24h）
2. 前端将 URL 保存到本地存储/页面状态
3. 25 小时后用户刷新页面
4. 使用过期 URL 访问 → 403 Forbidden
```

#### 场景 B：嵌入到邮件/通知

```
1. 系统发送邮件，附件链接使用预签名 URL
2. 用户 3 天后才打开邮件
3. 点击链接 → 签名已过期，无法访问
```

#### 场景 C：长期分享需求

```
1. 管理员生成报告链接分享给其他用户
2. 其他用户一周后尝试访问
3. URL 已过期，无法获取文件
```

### 6.3 规避方案

| 方案 | 实现方式 | 适用场景 |
|-----|---------|---------|
| 动态刷新 | 每次需要访问时重新调用 `presignGetUrl()` | 前端应用内访问 |
| 延长有效期 | 传入较大的 `expirationSeconds` | 短期分享（如 7 天） |
| 公开访问 | 设置 `enablePublicAccess=true` | 非敏感文件 |
| 自建代理 | 通过后端接口鉴权后再生成签名 | 完全可控的访问控制 |

---

## 七、业务记录删除与文件清理机制

### 7.1 文件管理模块的删除流程

**直接删除文件记录**（文件管理界面）：

```
DELETE /infra/file/delete?id=xxx
           │
           ▼
    FileService.deleteFile(id)
           │
     ┌─────┴─────┐
     ▼           ▼
  物理删除     删除数据库
  文件         FileDO 记录
```

**代码**：`FileServiceImpl.java:174-185`

### 7.2 业务模块引用文件的情况

**问题分析**：

假设业务场景：

```
订单表 (order)
├── id
├── attachment_url (存储 FileDO.url)
└── ...
```

当删除订单时：

```java
// 典型业务代码（假设）
public void deleteOrder(Long orderId) {
    OrderDO order = orderMapper.selectById(orderId);
    // ❌ 只删除业务记录
    orderMapper.deleteById(orderId);
    // ❌ 没有清理 order.getAttachmentUrl() 对应的文件
}
```

### 7.3 垃圾文件风险

| 场景 | 是否清理文件 | 结果 |
|-----|------------|------|
| 通过文件管理界面删除 | ✅ 是 | 物理文件被删除 |
| 业务模块删除引用记录 | ❌ 否 | 孤儿文件堆积 |
| 文件被多个业务引用，删一个 | ❌ 难处理 | 误删或残留 |

**设计缺陷**：
- FileDO 没有被引用计数机制
- 业务模块通常只存 URL，不存 fileId
- 没有统一的文件生命周期管理
- 缺少定时清理孤立文件的机制

---

## 八、安全问题分析：通过旧 URL 访问已删除私密附件

### 8.1 攻击场景构造

#### 前置条件

1. **系统配置**：使用 S3 存储，`enablePublicAccess=false`（私有桶）
2. **业务场景**：合同管理系统，支持上传私密合同文件
3. **存储方式**：合同表只存 `fileUrl`，不存 `fileId`

#### 攻击流程图

```
时间线 T0：正常上传
┌─────────────────────────────────────────────────────────────┐
│  管理员上传合同文件 contract-2024.pdf                         │
│         │                                                    │
│         ▼                                                    │
│  生成预签名 URL（24h 有效）                                    │
│  https://bucket.oss.com/20260510/contract-2024.pdf?signature │
│         │                                                    │
│         ▼                                                    │
│  保存到合同表：                                                │
│  contract.url = 上传时生成的完整 URL（带签名参数）              │
│         │                                                    │
│         ▼                                                    │
│  FileDO 记录：                                                │
│  ├── path = 20260510/contract-2024.pdf                       │
│  ├── url = https://bucket.oss.com/20260510/contract-2024.pdf │
│  └── configId = 1 (S3 私有配置)                               │
└─────────────────────────────────────────────────────────────┘

时间线 T1：攻击者获取 URL
┌─────────────────────────────────────────────────────────────┐
│  攻击者通过某种方式获取到 URL（如日志泄露、浏览器历史、XSS 等）  │
│  获取的 URL 可能是：                                          │
│  1. 前端展示的完整 URL（带当前签名）                           │
│  2. 数据库中保存的 URL（FileDO.url，不带签名参数）             │
│  3. 业务表中保存的 URL（可能是带签名或不带签名的）              │
└─────────────────────────────────────────────────────────────┘

时间线 T2：业务删除（漏洞触发点）
┌─────────────────────────────────────────────────────────────┐
│  管理员删除该合同记录                                         │
│         │                                                    │
│         ▼                                                    │
│  业务代码执行：                                                │
│  contractMapper.deleteById(contractId);                      │
│  // ❌ 没有清理关联文件！                                      │
│         │                                                    │
│         ▼                                                    │
│  当前状态：                                                    │
│  ✅ 合同记录已删除（业务上认为文件已"删除"）                     │
│  ✅ FileDO 记录仍然存在                                       │
│  ✅ S3 上的物理文件仍然存在                                   │
│  ❌ 没有任何清理动作                                          │
└─────────────────────────────────────────────────────────────┘

时间线 T3：攻击者访问（漏洞利用）
┌─────────────────────────────────────────────────────────────┐
│  情况 A：攻击者获取的是基础 URL（不带签名）                     │
│  URL: https://bucket.oss.com/20260510/contract-2024.pdf     │
│         │                                                    │
│         ▼                                                    │
│  私有桶直接访问 → 403 Forbidden ✅ 正常被拒绝                 │
│                                                             │
│  情况 B：攻击者获取的是最新的预签名 URL                         │
│  （通过其他途径，如系统漏洞、内部泄露）                         │
│         │                                                    │
│         ▼                                                    │
│  如果签名未过期 → 200 OK ❌ 文件泄露！                        │
│  如果签名已过期 → 403 ✅ 正常被拒绝                           │
│                                                             │
│  情况 C：更严重的问题                                          │
│  如果攻击者知道 path 结构，可以：                              │
│  1. 调用任意需要鉴权的接口获取新签名                           │
│  2. 或者如果存在未授权访问漏洞，直接构造访问                   │
└─────────────────────────────────────────────────────────────┘
```

### 8.2 核心问题点

#### 问题 1：业务删除 ≠ 文件删除

```
用户认知："我删除了合同记录" = "文件已被删除，不可访问"
实际情况："合同记录已删除" ≠ "物理文件已删除"
```

#### 问题 2：FileDO 记录独立存在

```
业务表（合同）  ──引用──▶  FileDO  ──关联──▶  物理文件
     │                      │                    │
     ▼                      ▼                    ▼
   被删除                 仍然存在              仍然存在
```

#### 问题 3：URL 泄露的多种途径

| 泄露途径 | 说明 |
|---------|------|
| 浏览器历史 | 用户曾访问过该 URL |
| 前端日志 | console.log、错误上报 |
| 网络代理 | 公司代理、抓包工具 |
| 数据库注入 | 如果存在 SQL 注入可查 FileDO 表 |
| 日志泄露 | 应用日志、Nginx 访问日志 |
| XSS 攻击 | 通过脚本获取页面上的 URL |
| 内部人员 | 离职员工保留的链接 |

### 8.3 攻击场景具体化

#### 场景一：员工离职后的未授权访问

```
1. 员工 A 在职期间负责合同管理，频繁访问合同文件
2. 员工 A 的浏览器历史/收藏夹保存了多个合同 URL
3. 员工 A 离职，但系统只是禁用账号
4. 员工 B 删除了员工 A 曾经处理的某份合同（业务记录）
5. 员工 A 使用家里电脑，从浏览器历史访问该 URL
6. 如果是公开桶或签名未过期 → ❌ 成功访问已"删除"的私密文件
```

#### 场景二：供应链/第三方泄露

```
1. 合同 URL 被包含在邮件通知中发送给供应商
2. 供应商的邮件系统被黑客入侵
3. 黑客获取到 URL 列表
4. 合同到期后，公司在系统中删除了该合同记录
5. 黑客尝试访问 URL → ❌ 文件仍然存在
```

#### 场景三：公开桶配置错误

```
1. 初始配置：私有桶（enablePublicAccess=false）
2. 某次运维操作：误将 S3 桶策略改为公开
3. 或者：创建新配置时勾选了公开访问，并设为 master
4. 结果：所有上传的文件变为公开可访问
5. 即使后续删除业务记录，只要知道 path 就能访问
```

### 8.4 风险等级评估

| 维度 | 评估 | 说明 |
|-----|------|------|
| 影响范围 | 中高 | 所有使用文件上传且只存 URL 的业务 |
| 利用难度 | 中 | 需要先获取 URL，但 URL 泄露途径多 |
| 隐蔽性 | 高 | 业务记录已删除，管理员认为文件已清理 |
| 数据敏感度 | 高 | 合同、发票、个人信息等私密文件 |
| 合规风险 | 高 | 违反数据删除要求（GDPR、个人信息保护法） |

### 8.5 修复建议

#### 建议一：建立文件引用关系

```java
// 方案 A：业务表存储 fileId 而非 url
public class ContractDO {
    private Long attachmentFileId;  // 存 ID，不存 URL
    // 访问时动态通过 FileService 生成 URL
}

// 方案 B：建立关联表
@TableName("biz_file_reference")
public class FileReferenceDO {
    private Long fileId;           // FileDO.id
    private String bizType;        // 业务类型：contract/order
    private Long bizId;            // 业务记录 ID
}
```

#### 建议二：业务删除时清理文件

```java
public void deleteContract(Long contractId) {
    ContractDO contract = contractMapper.selectById(contractId);
    
    // 1. 清理关联文件（如果是最后一个引用）
    if (contract.getAttachmentFileId() != null) {
        // 检查是否有其他业务引用该文件
        if (!fileReferenceService.hasOtherReferences(
                contract.getAttachmentFileId(), "contract", contractId)) {
            fileService.deleteFile(contract.getAttachmentFileId());
        }
    }
    
    // 2. 删除业务记录
    contractMapper.deleteById(contractId);
}
```

#### 建议三：定时清理孤儿文件

```java
@Scheduled(cron = "0 0 2 * * ?")  // 每天凌晨 2 点
public void cleanupOrphanFiles() {
    // 1. 找出没有被任何业务引用的 FileDO
    // 2. 或者：找出 deleted=true 但文件还存在的
    // 3. 执行物理删除
}
```

#### 建议四：增强私有桶访问控制

```yaml
# S3 配置建议
yudao:
  file:
    s3:
      enablePublicAccess: false      # 私密文件必须私有
      # 通过 CDN/WAF 增加访问控制层
      domain: https://cdn.example.com  # 经过 CDN
```

#### 建议五：URL 访问审计

```java
// 在文件下载接口增加审计日志
@GetMapping("/{configId}/get/**")
public void getFileContent(...) {
    // 记录：谁、何时、访问了哪个文件
    auditLogService.logFileAccess(userId, path, request.getRemoteAddr());
}
```

---

## 九、总结

### 9.1 关键发现

| 问题点 | 严重程度 | 说明 |
|-------|---------|------|
| 业务删除不清理文件 | 🔴 高 | 孤儿文件可能泄露敏感数据 |
| 私有桶签名 URL 24h 过期 | 🟡 中 | 缓存/邮件场景需注意 |
| 本地存储接口 PermitAll | 🟡 中 | 知道 URL 即可访问 |
| 无引用计数机制 | 🟡 中 | 多业务引用同文件时难管理 |
| URL 存业务表而非 fileId | 🟡 中 | 不利于统一管理和清理 |

### 9.2 最佳实践建议

1. **私密文件使用私有桶**：`enablePublicAccess=false`
2. **存储 fileId 而非 url**：访问时动态生成签名
3. **删除业务时清理文件**：建立引用计数或关联表
4. **定时清理孤儿文件**：定时任务扫描未引用文件
5. **敏感文件访问审计**：记录所有文件访问日志
6. **重要文件加密**：上传前加密，下载后解密

### 9.3 代码导航速查

| 功能 | 文件路径 |
|-----|---------|
| 文件实体 | `yudao-module-infra/dal/dataobject/file/FileDO.java` |
| 配置实体 | `yudao-module-infra/dal/dataobject/file/FileConfigDO.java` |
| 上传控制器 | `yudao-module-infra/controller/admin/file/FileController.java` |
| 上传服务 | `yudao-module-infra/service/file/FileServiceImpl.java` |
| 本地存储客户端 | `yudao-module-infra/framework/file/core/client/local/LocalFileClient.java` |
| S3 存储客户端 | `yudao-module-infra/framework/file/core/client/s3/S3FileClient.java` |
| 文件类型工具 | `yudao-module-infra/framework/file/core/utils/FileTypeUtils.java` |
| 全局配置 | `yudao-server/src/main/resources/application.yaml:14-16` |
