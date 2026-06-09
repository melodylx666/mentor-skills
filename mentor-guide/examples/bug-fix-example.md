# Bug 修复示例：Spring Boot 图片上传 OOM

> 本示例展示 mentor-guide 工作流在处理 Java 生产环境 OOM 问题时的完整应用过程。

---

## 第一阶段：问题诊断

### 问题复述

生产环境的 Spring Boot 应用（用户头像上传服务）每隔 2-3 天出现一次 `java.lang.OutOfMemoryError: Java heap space`，重启后恢复，但几天后再次出现。堆内存配置为 512MB，无明显流量高峰。

### 问题分类

- **类型**：运行时异常 / 性能问题（内存泄漏）
- **环境**：Spring Boot 2.7.x / OpenJDK 11 / 4C8G 云服务器 / 堆内存 512MB
- **严重程度**：**阻塞** — 导致服务不可用，影响用户上传功能

### 初步判断

> （以下为基于经验的推测，需通过搜索和验证确认）

- **推测一**：图片上传后未释放临时资源（如 `MultipartFile.transferTo()` 后临时文件残留）→ 可能性中等
- **推测二**：每次上传将整个图片加载到内存再处理 → 可能性较高
- **推测三**：某个第三方库存在内存泄漏 → 可能性较低

### 需要确认的信息

1. 堆转储（Heap Dump）是否可用？
2. 上传图片的平均大小和最大大小是多少？
3. OOM 发生时 JVM 参数配置详情？

---

## 第二阶段：方案搜索

### 搜索过程记录

#### ① 关键词搜索 — 定位根因方向

```
搜索词：Spring Boot MultipartFile OutOfMemoryError Java heap space
搜索平台：GitHub Issues + Stack Overflow
```

**搜索结果摘要：**

| 信源 | 内容 | 可信度 |
|------|------|--------|
| GitHub Spring Framework Issues #25245 | 大量用户报告 `@RequestParam MultipartFile` 在并发上传大文件时导致 OOM | **A-权威**（官方仓库） |
| Stack Overflow QA（高赞） | 指出 `MultipartFile` 默认将整个文件加载到内存，大文件场景需改用 `InputStream` | **B-可信**（高赞答案） |
| Baeldung Tutorial | Spring Boot 文件上传大小默认配置为 1MB，超过后抛出异常而非触发 OOM | **C-参考**（技术教程站） |

#### ② 深入搜索 — 定位根因

```
搜索词：Spring Boot multipart max-file-size vs OutOfMemoryError
```

**发现关键信息**：

- `spring.servlet.multipart.max-file-size` 默认值 = 1MB（Spring Boot 2.x）
- 但应用中此值被设置为 `500MB`，导致允许超大文件上传
- Spring 的 `StandardMultipartFile` 将整个文件内容加载到 `byte[]` 中，500MB 文件直接导致堆内存压力
- 并发上传时，多个大文件同时驻留内存，迅速填满 512MB 堆空间

```
搜索词：Spring Boot large file upload streaming CommonsMultipartFile
```

**发现替代方案**：

- `CommonsMultipartFile` 可配合 `Commons FileUpload` 的 `DiskFileItem` 实现临时文件写入，避免全量内存加载
- 更推荐的方式：使用 `InputStream` 直接获取流式数据，配合 `FileCopyUtils` 流式写入磁盘

#### ③ 交叉验证

```
搜索词：Spring Boot streaming file upload best practice
搜索平台：GitHub Issues + Spring Official Docs
```

**验证结论**：

- Spring 官方文档明确推荐：处理大文件时应使用 `InputStream` 而不是 `MultipartFile`
- GitHub 上 Spring Boot 示例项目使用 `InputStream` 方式处理超过 10MB 的文件上传
- Netflix / 阿里云技术博客均有类似案例记录

---

## 第三阶段：解决方案

### 问题根因

**根因确认（信源交叉验证通过）：**

Spring Boot 默认使用 `StandardMultipartFile` 实现，它会在调用 `getBytes()` 或 `getInputStream()` 前将**整个文件内容加载到 JVM 堆内存**中。应用中将 `max-file-size` 设置为 500MB，且未做文件大小前置校验。当用户上传大图片（>100MB）时，单个请求即可占用近 20% 的堆空间；并发 5 个以上大文件上传请求时，堆内存瞬间耗尽，触发 OOM。

> 信源引用：
> - [GitHub Spring Framework #25245](https://github.com/spring-projects/spring-framework/issues/25245) — 官方确认此行为（**A-权威**）
> - [Spring Boot Doc - File Upload Limits] — 配置说明（**A-权威**）
> - [Stack Overflow - MultipartFile OOM] — 社区验证（**B-可信**）

### 推荐方案

#### 方案一：添加文件大小限制 + 流式处理（推荐 ✅）

- **适用场景**：所有使用 `MultipartFile` 处理文件上传的场景
- **操作步骤**：

**第 1 步：收紧配置文件中的上传大小限制**

```yaml
# application.yml
spring:
  servlet:
    multipart:
      max-file-size: 10MB        # 单文件最大 10MB（原为 500MB）
      max-request-size: 20MB     # 请求体最大 20MB
```

**第 2 步：在 Controller 层添加前置校验**

```java
@PostMapping("/avatar/upload")
public ResponseEntity<?> uploadAvatar(@RequestParam("file") MultipartFile file) {
    // 前置校验
    if (file.isEmpty()) {
        return ResponseEntity.badRequest().body("文件不能为空");
    }
    if (file.getSize() > 10 * 1024 * 1024) {
        return ResponseEntity.badRequest().body("文件大小不能超过 10MB");
    }

    // 使用 InputStream 流式处理
    try (InputStream inputStream = file.getInputStream()) {
        String filePath = fileStorageService.store(inputStream, file.getOriginalFilename());
        return ResponseEntity.ok(Map.of("path", filePath));
    } catch (IOException e) {
        return ResponseEntity.internalServerError().body("上传失败");
    }
}
```

**第 3 步：存储层使用流式写入**

```java
// FileStorageService.java
public String store(InputStream inputStream, String originalFilename) throws IOException {
    String targetPath = generateTargetPath(originalFilename);
    // 使用 Files.copy 流式写入，不占用堆内存
    Files.copy(inputStream, Path.of(targetPath), StandardCopyOption.REPLACE_EXISTING);
    return targetPath;
}
```

- **优点**：
  - 从根本上解决大文件上传导致的 OOM
  - 流式处理仅使用固定大小的缓冲区（默认 8KB），与文件大小无关
  - 前置校验提供更好的用户体验
- **缺点**：
  - 需要修改现有 Controller 和 Service 代码
  - 依赖 `InputStream` 处理不支持随机访问（但上传场景不需要）

- **参考来源**：
  - [Spring Boot Reference Doc - File Upload](https://docs.spring.io/spring-boot/docs/current/reference/html/web.html#web.servlet.embedded-container.multipart)（**A-权威**）
  - [Spring Framework Issue #25245 - multipart file memory usage](https://github.com/spring-projects/spring-framework/issues/25245)（**A-权威**）
  - [Baeldung - MultipartFile to InputStream](https://www.baeldung.com/spring-multipartfile-to-inputstream)（**C-参考**）

#### 方案二：使用 Nginx 前置代理做大小限制（补充方案）

- **适用场景**：无法修改应用代码，或者需要多层防护
- **操作步骤**：

```nginx
# nginx.conf
server {
    listen 80;
    client_max_body_size 10M;  # 限制请求体大小，超出直接返回 413
    
    location /api/upload {
        proxy_pass http://backend;
        proxy_request_buffering off;  # 关闭请求体缓冲，流式转发
    }
}
```

- **优点**：不侵入应用代码，在网关层拦截大文件
- **缺点**：仅做大小限制，不解决应用内部流式处理问题

- **参考来源**：
  - [Nginx Doc - client_max_body_size](http://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size)（**A-权威**）
  - [Nginx Doc - proxy_request_buffering](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_request_buffering)（**A-权威**）

### 预防建议

1. **JVM 层面**：生产环境配置 `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/path/to/dumps`，确保下次 OOM 时能拿到堆转储做根因分析
2. **监控层面**：配置堆内存使用率告警（如 >80% 持续 5 分钟），在 OOM 发生前介入
3. **CI/CD 层面**：在 CI 中加入安全配置检查 —— 扫描 `application.yml` 中 `max-file-size` 是否超过合理阈值
4. **代码审查层面**：所有涉及文件上传的 PR 必须审查是否存在全量加载到内存的操作
5. **压测层面**：上线前对文件上传接口做极限压测，验证在大文件 + 并发场景下的内存表现

---

## 第四阶段：方法论总结

### 本次问题的排查路径

```
1. 问题表面分析 → OOM，定位到是文件上传服务
2. 堆内存分析 → 怀疑大对象分配（启动 -XX:+HeapDumpOnOutOfMemoryError 后复现）
3. 堆转储分析 → 发现 byte[] 占堆内存 85%，全来自 MultipartFile
4. 搜索验证 → GitHub Issues 确认 Spring MultipartFile 的全量加载行为
5. 配置检查 → 发现 max-file-size 被设置为 500MB
6. 制定方案 → 收紧限制 + 改用流式处理
7. 验证方案 → 压测确认内存稳定
```

### Java OOM 通用排查路径

```
遇到 OOM Error
    │
    ├─ 是否有 Heap Dump？
    │   ├─ 有 → 用 MAT / JProfiler 分析：最大对象是谁？什么 GC Root？
    │   └─ 无 → 添加 -XX:+HeapDumpOnOutOfMemoryError 后复现
    │
    ├─ 是内存泄漏还是内存溢出？
    │   ├─ 内存泄漏 → 对象无法被 GC（如 ThreadLocal 未清理、连接池泄漏）
    │   └─ 内存溢出 → 对象太大/太多，但 GC 正常（如大文件、大量请求）
    │
    ├─ 常见模式：
    │   ├─ byte[] 占比高 → 文件/图片处理、JSON 序列化、加密解密
    │   ├─ char[] 占比高 → 字符串处理、日志输出
    │   ├─ HashMap/ArrayList 增长 → 数据量超出预期
    │   └─ Thread 数量多 → 线程栈占用非堆内存
    │
    └─ 解决后，加上监控和告警，防止复现
```

### 推荐的进一步学习资源

1. **[Java Memory Management - Oracle Docs](https://docs.oracle.com/en/java/javase/11/gctuning/)** — JVM 内存管理与 GC 调优官方指南（**A-权威**）
2. **[Spring Boot Multipart File Upload Guide](https://spring.io/guides/gs/uploading-files/)** — Spring 官方文件上传教程（**A-权威**）
3. **[Eclipse MAT 教程](https://www.eclipse.org/mat/)** — 堆转储分析工具入门（**B-可信**）
4. **[Java 内存泄漏排查实战 - 美团技术博客](https://tech.meituan.com/)** — 大厂实战经验（**B-可信**）

---

## 信源汇总

| 序号 | 信源 | 类型 | 可信度 |
|------|------|------|--------|
| 1 | [Spring Framework Issue #25245](https://github.com/spring-projects/spring-framework/issues/25245) | 官方 Issues | **A-权威** |
| 2 | [Spring Boot Reference - File Upload](https://docs.spring.io/spring-boot/docs/current/reference/html/web.html) | 官方文档 | **A-权威** |
| 3 | [Stack Overflow - MultipartFile OOM] | 社区问答 | **B-可信** |
| 4 | [Baeldung - MultipartFile to InputStream](https://www.baeldung.com/spring-multipartfile-to-inputstream) | 技术教程 | **C-参考** |
| 5 | [Nginx - client_max_body_size](http://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size) | 官方文档 | **A-权威** |
| 6 | [美团技术博客 - Java 内存泄漏排查] | 大厂博客 | **B-可信** |

> **最低信源要求检查** ✅：Bug 修复 ≥ 1 个 B 级 → 满足（实际有 3 个 A 级 + 2 个 B 级）

---

*本示例为模拟案例，旨在展示 mentor-guide 工作流的应用方式。实际场景中信源链接为真实可查内容。*
