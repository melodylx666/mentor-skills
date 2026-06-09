# 通用调试排查框架

## 1. 信息收集阶段

### 1.1 收集完整报错信息
- **堆栈信息**：捕获完整异常堆栈（Exception stack trace），不要只看第一行
  - Java：`e.printStackTrace()` 或 `log.error("msg", e)`
  - 注意查看 `Caused by` 链，根因通常在链的最内层
- **日志上下文**：收集报错前后至少 20 行日志，关注 WARN/ERROR 级别
- **截图描述**：如果是 UI 问题，提供截图和操作步骤

### 1.2 确认复现条件
- **输入数据**：什么输入触发了问题？是特定值还是任意值？
- **运行环境**：
  - 操作系统及版本
  - JDK / 运行时版本
  - 框架版本（Spring Boot, Flink 等）
  - 是否容器化（Docker 镜像版本、资源限制）
- **触发频率**：必现 / 偶现（概率多少）？偶现问题需记录触发模式

### 1.3 确认最近改动
- **代码变更**：最近提交了哪些代码？使用 `git log --oneline -10` 或 `git diff HEAD~1`
- **依赖升级**：检查 `pom.xml` / `build.gradle` 中是否有版本变更
- **配置变更**：对比配置文件的变更（`application.yml`, `flink-conf.yaml` 等）
- **基础环境变更**：是否有 JVM 升级、操作系统补丁、网络策略变更

---

## 2. 根因定位方法

### 2.1 二分法
> 适用于代码规模较大、不确定问题范围时

**做法**：注释/屏蔽一半代码，观察问题是否复现。不断缩小范围，直到锁定具体代码段。

```
原始代码：A → B → C → D → E → F（出问题）
第一步：屏蔽 D→E→F，问题消失 → 问题在 D→E→F 范围内
第二步：只屏蔽 D，问题仍在 → 问题不在 D
第三步：只屏蔽 E，问题消失 → 问题在 E
```

### 2.2 最小复现法
> 适用于需要向他人求助或写 issue 时

**做法**：从完整代码中逐步删除无关部分（业务逻辑、配置、外部调用），保留**能复现问题的最少代码**。

**示例**：
```java
// 逐步删除无关逻辑后的最小复现
public class MinimalReproducer {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>();
        list.add("test");
        // 删除了业务逻辑，只保留关键操作
        list.get(10);  // IndexOutOfBoundsException
    }
}
```

### 2.3 对比法
> 适用于环境相关问题

**做法**：逐一对比正常环境与异常环境的差异项。

| 对比项 | 正常环境 | 异常环境 |
|--------|----------|----------|
| JDK 版本 | 1.8.0_292 | 1.8.0_202 |
| 堆内存 | -Xmx2g | -Xmx512m |
| 依赖版本 | fastjson 1.2.70 | fastjson 1.2.60 |
| 操作系统 | CentOS 7.9 | CentOS 7.6 |

### 2.4 日志增强法
> 适用于复杂流程中的偶现问题

**做法**：在关键路径添加临时日志，缩小排查范围。

```java
// 增强前
public Order processOrder(OrderDTO dto) {
    Order order = convert(dto);
    validate(order);
    save(order);
    notify(order);
    return order;
}

// 增强后
public Order processOrder(OrderDTO dto) {
    log.info("[DEBUG] processOrder start, dto={}", dto);
    Order order = convert(dto);
    log.info("[DEBUG] after convert, order={}", order);
    validate(order);
    log.info("[DEBUG] after validate, result={}", order.getStatus());
    save(order);
    log.info("[DEBUG] after save, orderId={}", order.getId());
    notify(order);
    log.info("[DEBUG] after notify, done");
    return order;
}
```

---

## 3. 分类排查指南

### 3.1 编译报错
按以下优先级排查：

1. **依赖冲突** — 同一个依赖存在多个版本
   - Maven：`mvn dependency:tree -Dincludes=<groupId>:<artifactId>`
   - Gradle：`gradle dependencies --configuration compileClasspath`
   - 检查是否有多版本 jar 被同时加载

2. **语法错误** — 编译器直接提示
   - 泛型擦除、类型不匹配
   - Lambda 表达式中的变量捕获
   - 未捕获的受检异常

3. **版本不兼容** — API 在升级后被移除或变更
   - Spring Boot 2.x → 3.x：`javax.*` → `jakarta.*`
   - JDK 版本升级导致 `sun.misc.*` 不可用

### 3.2 运行时异常

| 异常类型 | 常见原因 | 排查命令 / 思路 |
|----------|----------|-----------------|
| **NullPointerException** | 未判空、未初始化 | 看堆栈行号，确认哪个变量为 null |
| **ClassCastException** | 类型强转错误 | 确认对象真实类型（`getClass()`） |
| **IndexOutOfBoundsException** | 集合/数组越界 | 检查索引计算逻辑 |
| **OutOfMemoryError** | 堆/元空间/直接内存不足 | `jmap -heap` / `jstat -gc` |
| **StackOverflowError** | 无限递归 | 检查递归调用是否有正确的终止条件 |
| **ConcurrentModificationException** | 遍历时修改集合 | 使用 `CopyOnWriteArrayList` 或同步块 |
| **NoSuchMethodError** | 编译期与运行期 jar 版本不一致 | `mvn dependency:tree` 排查依赖冲突 |

### 3.3 性能问题

**排查路径**：慢查询 → 内存泄漏 → 锁竞争 → IO 瓶颈

1. **慢查询**
   ```sql
   -- MySQL 开启慢查询日志
   SET GLOBAL slow_query_log = ON;
   SET GLOBAL long_query_time = 1;
   -- 查看执行计划
   EXPLAIN SELECT * FROM orders WHERE user_id = ?;
   ```
   - 确认索引是否命中（`type` 是否为 `ref`/`range`/`const`，避免 `ALL`）
   - 检查是否有多余的 `JOIN` 或子查询

2. **内存泄漏**
   ```bash
   # 查看堆使用概况
   jstat -gcutil <pid> 1000 10
   
   # 生成堆转储快照
   jmap -dump:live,format=b,file=heap.hprof <pid>
   
   # 使用 VisualVM 或 MAT 分析 heap.hprof
   ```
   - 关注 `Full GC` 频率和时长
   - 检查是否有对象无法被 GC 回收（如 ThreadLocal 未移除、静态集合无限增长）

3. **锁竞争**
   - `jstack <pid>` 导出线程栈，搜索 `BLOCKED` / `WAITING` 状态的线程
   - 使用 Arthas：`thread -b` 查看当前阻塞的线程
   - 减少锁粒度：读写锁、分段锁、CAS 替代

4. **IO 瓶颈**
   ```bash
   # Linux 查看磁盘 IO
   iostat -x 1
   # 查看网络 IO
   netstat -s
   ```
   - 确认磁盘利用率（`%util`）是否接近 100%
   - 关注网络重传率（`retransmit`）是否异常

### 3.4 网络问题

| 症状 | 可能原因 | 排查命令 |
|------|----------|----------|
| 连接超时 | 防火墙拦截 / 路由不可达 | `telnet <host> <port>` / `curl -v` |
| 连接池耗尽 | 连接未释放 / QPS 过高 | `netstat -an \| grep <port> \| wc -l` |
| DNS 解析失败 | DNS 配置错误 / 域名不存在 | `nslookup <domain>` / `dig <domain>` |
| 请求被重置 | 防火墙 / 对端主动关闭 | `tcpdump -i eth0 host <ip>` |

---

## 4. 调试工具速查

### 4.1 JVM 工具

| 工具 | 用途 | 常用命令示例 |
|------|------|-------------|
| **jps** | 查看 Java 进程 | `jps -lvm` |
| **jstack** | 查看线程堆栈 | `jstack <pid> > thread_dump.txt` |
| **jmap** | 堆内存分析 | `jmap -heap <pid>` / `jmap -dump:live,format=b,file=heap.hprof <pid>` |
| **jstat** | GC 监控 | `jstat -gcutil <pid> 2000 10`（每 2 秒输出一次，共 10 次） |
| **jinfo** | JVM 配置查看 | `jinfo -flags <pid>` |
| **VisualVM** | 可视化监控 | 安装 JDK 自带或独立版，支持 CPU/内存/线程实时监控 |
| **Arthas** | 在线诊断 | `curl -O https://arthas.aliyun.com/arthas-boot.jar && java -jar arthas-boot.jar` |
| | | `watch <class> <method> '{params,returnObj}' -x 2` |
| | | `trace <class> <method>`（方法调用链路耗时）|
| | | `dashboard`（实时面板）|

**Arthas 常用场景**：
```bash
# 查看方法调用耗时
trace com.example.service.OrderService createOrder

# 查看方法入参和返回值
watch com.example.service.OrderService createOrder "{params,returnObj}" -x 2

# 热替换类（紧急修复用）
redefine /tmp/OrderService.class
```

### 4.2 Linux 工具

| 工具 | 用途 | 常用命令示例 |
|------|------|-------------|
| **strace** | 系统调用追踪 | `strace -p <pid> -t -f -o strace.log` |
| | | `strace -e trace=open,read,write -p <pid>` |
| **top** / **htop** | 进程资源监控 | `top -H -p <pid>`（查看线程级 CPU） |
| **iostat** | 磁盘 IO 监控 | `iostat -x 1` |
| | | 关注 `await`（IO 响应时间）、`%util`（磁盘利用率） |
| **netstat** | 网络连接状态 | `netstat -anp \| grep <port>` |
| | | `netstat -s`（协议级统计） |
| | | `ss -tan`（更快速的替代方案） |
| **tcpdump** | 抓包分析 | `tcpdump -i eth0 host <ip> and port <port> -w capture.pcap` |
| **dmesg** | 内核日志 | `dmesg -T \| tail -50`（查看 OOM Killer 等） |
| **lsof** | 打开文件查看 | `lsof -p <pid>` |
| | | `lsof -i :<port>`（查看端口占用） |

### 4.3 数据库工具

| 工具 / 命令 | 用途 | 示例 |
|-------------|------|------|
| **EXPLAIN** | 查看执行计划 | `EXPLAIN SELECT * FROM orders WHERE status = 'PENDING';` |
| | | 关注 `type`（ALL 表示全表扫描）、`rows`（预估扫描行数）、`Extra`（Using filesort 表示需优化） |
| **slow query log** | 慢查询日志 | `SET GLOBAL slow_query_log = ON; SET GLOBAL long_query_time = 1;` |
| | | 分析工具：`mysqldumpslow -s t -t 10 /var/lib/mysql/slow.log` |
| **SHOW PROCESSLIST** | 查看当前连接 | `SHOW FULL PROCESSLIST;` |
| | | 关注 `Time`（执行时间）、`State`（Sending data 表示正在扫描）、`Info`（执行的 SQL） |
| **performance_schema** | 性能诊断（MySQL 5.7+） | 查看等待事件、锁等待、IO 统计 |
| **pg_stat_activity** | PostgreSQL 会话 | `SELECT * FROM pg_stat_activity WHERE state = 'active';` |

---

## 附录：调试方法论总结

```
遇到问题时 → 不要凭感觉修改代码
     ↓
收集信息 → 整理问题全貌
     ↓
提出假设 → 基于证据，不是猜测
     ↓
验证假设 → 用实验确认（改代码、加日志、查监控）
     ↓
定位根因 → 找到根本原因，不是表象
     ↓
修复验证 → 修复后确认问题不再复现
     ↓
复盘总结 → 记录根因和排查过程，防止同类问题再次发生
```

### 常见误区

1. **跳过复现**：不能稳定复现就动手改代码 → 先确认复现条件
2. **过度分析**：从头到尾读代码找 bug → 用日志/断点缩小范围
3. **忽略环境**：只看代码忽略配置、依赖、OS 差异 → 对比法排查
4. **一次改多处**：同时改了 N 个地方，问题消失但不知道哪处有效 → 一次只改一处
