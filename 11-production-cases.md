
[⬅️ 上一章](10-offheap-container.md) · [📖 返回目录](README.md) · [➡️ 下一章](12-jvm-flags.md)
# 11 · 生产案例集：15 个真实故障复盘

> **本章定位**：前 10 章的方法论落到真实战场。每个案例按「症状→排查→根因→修复→复盘要点」组织，全部来自公开技术社区的一线实践（美团、阿里云、HeapDump 社区、个人博客），出处逐一注明。读案例的目的是**建立「症状→第一反应」的条件反射**，面试讲案例时能体现完整排查链路。

---

### 11.1 案例索引

| # | 案例 | 类别 | 对应章节 |
|---|------|------|---------|
| 1 | 正则回溯打满 CPU | CPU | 07 §7.3 |
| 2 | 大量线程 BLOCKED，日志框架锁风暴 | 锁竞争 | 08 §8.5 |
| 3 | 无界队列 OOM：FixedThreadPool 堆积 | 线程池 | 08 §8.3 |
| 4 | 元空间持续上涨（Groovy 脚本引擎） | 类加载/元空间 | 09 §9.3 |
| 5 | 容器反复 OOMKilled 137 | 容器内存 | 10 §10.3 |
| 6 | Netty ByteBuf 泄漏导致 Direct OOM | 堆外 | 10 §10.5 |
| 7 | G1 Humongous 引发周期性长停顿 | GC | 06 §6.4 |
| 8 | 缓存无界膨胀致堆 OOM | 内存泄漏 | 04 |
| 9 | 下游 hang 死拖垮连接池（假死） | 线程假死 | 08 §8.6 |
| 10 | ThreadLocal 泄漏 + 类加载器无法卸载 | 泄漏/类加载 | 09 §9.3 |

> 案例细节持续补充中；以下为已核实出处的完整版。

---

### 11.2 案例 1：一个正则表达式引发的血案

**来源**：陈树义《一个正则表达式引发的血案》（腾讯云社区/知乎专栏，zhuanlan.zhihu.com/p/38229530）

- **症状**：CPU 接近 100%，dump 里 100+ 处堆栈全部指向同一个 `validateUrl` 方法。
- **排查**：导出线程 dump 定位到 URL 校验正则 → 单测复现：用问题 URL 匹配 `^([hH][tT]{2}[pP]://...)([A-Za-z0-9-~]+.)+...+$`，进程 CPU 飙到 91.4% → regex101 直接提示 catastrophic backtracking。
- **根因**：URL 含字符类未覆盖字符（`%5E` 的 `%` 等），嵌套贪婪组 `((...)+.)+` 回溯爆炸；每个请求卡死在校验上，线程堆积打满 CPU。
- **修复**：独占模式 `++` 消除回溯；上线前用 regex101 检测危险正则。
- **复盘要点**：补字符类只是打补丁——下次换个特殊字符还会复发。正则 DoS 要从模式本身消除，而不是修补输入。

---

### 11.3 案例 2：日志框架引发的锁风暴（BLOCKED 集群）

**来源**：HeapDump 社区与美团技术团队多起同类复盘综合

- **症状**：接口 RT 从 20ms 劣化到 2s+，CPU 不高；jstack 三连拍显示几十个业务线程全部 `BLOCKED (on object monitor)`，等待目标高度集中在同一个锁地址，持有者栈顶落在日志 Appender 的同步写盘方法里。
- **排查**：统计 `waiting to lock <0x…>` 目标分布（08 章 §8.5 脚本）→ 反查持锁线程：正在写一条超大报文日志到慢速磁盘（IO await 高）→ 所有经过该 logger 的请求排队。
- **根因**：同步 Appender 全局锁 × 慢磁盘 IO = 串行化所有业务线程。「锁内做 IO」是锁风暴的头号配方。
- **修复**：Appender 换异步（Log4j2 AsyncLogger，Disruptor 无锁队列）；大报文截断脱敏后再记；磁盘问题单独治理。
- **复盘要点**：看到大面积 BLOCKED 先问三件事——谁持有？持有多久？为什么这么久？答案通常指向「临界区里混进了 IO」。

---

### 11.4 案例 3：FixedThreadPool 无界队列堆积 OOM

**来源**：阿里《Java 开发手册》强制规约的典型事故原型 + HeapDump 社区复盘

- **症状**：批量处理服务运行数小时后 `OutOfMemoryError: Java heap space`；dump 直方图显示上千万个 `LinkedBlockingQueue$Node` 与待处理任务对象。
- **排查**：dump 支配树显示队列对象支配了 90%+ 堆 → 队列 size 数百万 → 线程池是 `Executors.newFixedThreadPool(8)`（无界队列）→ 追查任务耗时：上游依赖接口从 50ms 劣化到 5s，生产速率远超消费。
- **根因**：无界队列让「背压」彻底失效——下游一慢，内存变成蓄水池，max/拒绝策略形同虚设。OOM 只是最终表现，真凶是下游劣化无人知晓。
- **修复**：手动 new ThreadPoolExecutor + 有界队列（容量=可接受延迟×吞吐）+ AbortPolicy 快速失败 + 队列深度监控告警；下游加超时熔断。
- **复盘要点**：队列不是越大越好——**有界队列是系统的神经末梢**，满了会喊疼（拒绝/告警），无界队列只会闷声膨胀到死。

---

### 11.5 案例 4：Groovy 脚本引擎吃光元空间

**来源**：HeapDump 社区《Metaspace 内存泄漏》系列案例综合

- **症状**：规则引擎服务每两周左右 Metaspace OOM；FGC 后元空间只降一点点继续涨。
- **排查**：`jcmd GC.class_loader_stats` 发现数千个 GroovyClassLoader 实例、每个加载了几十个脚本类 → MAT Duplicate Classes 显示同名规则类重复上百次 → 对某个 GroovyClassLoader 做 Path to GC Roots：被业务静态 Map 缓存钉住（缓存 key 是规则 ID，value 连着旧加载器编译产物）。
- **根因**：每次规则热更新都新建加载器重新编译脚本，但旧版本被缓存引用无法回收——Class 钉住 Loader，Loader 钉住整套元数据。
- **修复**：规则更新时显式移除旧缓存项并释放旧加载器；Groovy 引擎设置脚本编译结果复用；MaxMetaspaceSize 封顶提前暴露问题。
- **复盘要点**：「动态生成类」的服务必须回答一个问题——**旧类和旧加载器由谁在什么时候释放？** 答不出来就是定时炸弹。

---

### 11.6 案例 5：容器反复 OOMKilled 137

**来源**：Kubernetes 社区与各大厂容器化迁移复盘综合

- **症状**：JVM 从物理机迁入 K8s 后每天被杀数次；应用日志干净没有任何 OOM 记录；`kubectl describe pod` 显示 `Last State: OOMKilled, Exit Code: 137`。
- **排查**：dmesg 确认 oom-killer 出手 → RSS 峰值 ≈ limit；账本核算：Xmx=limit×90%（照搬物理机习惯），加上元空间 200M + 栈 300M + CodeCache 240M + DirectBuffer，堆外直接溢出 → 且基础镜像是 JDK 8u111（≤8u131 不感知 cgroup，GC 线程按宿主机 64 核配置）。
- **根因**：双重问题——Xmx 占比过大挤掉堆外 + 老 JDK 感知错误核数导致线程/GC 过度配置。
- **修复**：升级 8u191+；`-XX:MaxRAMPercentage=75 -XX:MaxMetaspaceSize=256m -XX:MaxDirectMemorySize=512m`；limit 调整为峰值 RSS×1.25；working set 80% 告警。
- **复盘要点**：容器化不是「打包成镜像」就完了——**内存预算表必须重做，JDK 版本是第一检查项**。

---

### 11.7 案例 6：Netty ByteBuf 泄漏拖垮网关

**来源**：Netty 官方 leak detection 文档场景 + HeapDump 社区网关案例综合

- **症状**：API 网关运行 3~5 天后突发 `OutOfMemoryError: Direct buffer memory`；重启恢复，周期复发；RSS 缓慢爬升。
- **排查**：NMT 的 Internal 不动（Netty 自记账不进 NMT）→ Arthas watch PlatformDependent.usedDirectMemory 持续上涨 → 开 `-Dio.netty.leakDetection.level=paranoid` 后日志出现 LEAK 记录，RECENT 堆栈指向自定义 Handler：解析失败分支 return 时没有 release。
- **根因**：异常路径漏释放。正常请求都走 TailContext 自动释放，只有特定畸形报文触发 early-return 分支——低频路径的泄漏最难在测试期暴露。
- **修复**：该分支 finally 里 ReferenceCountUtil.release(msg)；CI 增加 Netty 泄漏检测回归用例（paranoid 模式跑压测）。
- **复盘要点**：引用计数的铁律是**所有路径都要 release，包括异常路径**。leak detection 平时就开 simple 档，别等出事才想起来。

---

### 11.8 案例 7：G1 Humongous 周期性长停顿

**来源**：美团技术团队《Java 中 9 个常见的 G1 问题》与社区 G1 调优复盘综合

- **症状**：每 30 分钟一次 RT 毛刺（200ms~1s）；GC 日志显示 Young GC 正常但偶发 `Pause Full (G1 Compaction Pause)`；humongous regions 分配频繁。
- **排查**：对齐毛刺时间点与定时任务执行时间吻合 → 该任务批量拉取大列表反序列化为超大对象数组 → 单对象 > Region 大小一半即 Humongous，直接进老年代并造成碎片 → 触发并发标记后紧跟 Full GC 压缩。
- **根因**：Humongous 对象绕过分代设计直达老年代，反复制造碎片。「大对象」不一定大——Region 4MB 时一个 2MB 数组就算。
- **修复**：任务分批处理把单对象降到 KB 级；`-XX:G1HeapRegionSize=16m` 应急调大；监控 humongous region 占比。
- **复盘要点**：G1 的性能问题先问三件事——**有没有大对象？Mixed 回收跟不跟得上？并发标记周期多密？** 周期性毛刺优先怀疑定时任务。

---

### 11.9 案例 8：本地缓存无界膨胀堆 OOM

**来源**：HeapDump 社区《MAT 分析堆溢出》典型案例

- **症状**：查询服务堆使用率缓慢上涨，FGC 能短暂回落但基线持续抬升，最终 heap space OOM。
- **排查**：两份 dump（间隔一天）MAT 直方图对比，某业务 DTO 对象数量翻倍增长 → 支配树指向静态 HashMap 缓存 → 代码审查确认缓存 key 含用户上下文参数，实际基数远超预期（原以为千级，实为百万级）且无淘汰策略。
- **根因**：「缓存」没有容量上限、没有过期、key 设计基数失控——本质是无界集合伪装成缓存。
- **修复**：换 Caffeine 设置 maximumSize + expireAfterWrite；监控缓存条数与命中率；容量评估写进设计文档。
- **复盘要点**：任何进程内缓存上线前必须回答三个数字——**最大多少条？每条多大？过期多久？** 答不出就不配上生产。

---

### 11.10 案例 9：下游 hang 死拖垮连接池（服务假死）

**来源**：美团技术团队超时治理文章 + 社区故障复盘综合

- **症状**：订单服务突然全量超时，进程存活、健康检查通过；jstack 三连拍显示全部工作线程 WAITING/RUNNABLE 卡在 HTTP client 的 socketRead0。
- **排查**：栈顶全部 socketRead0 且无 readTimeout 配置 → 下游依赖某外部接口因网络抖动 hang 住 → 本服务连接池 50 个连接被逐步借空 → 新请求全部阻塞在 acquire。
- **根因**：外呼无超时 × 连接池有限 = 故障传导链。下游 hang 住多久，本服务就死多久。
- **修复**：connectTimeout=1s / socketTimeout=3s 全量覆盖；连接池 maxWait 设上限；核心依赖加 Sentinel 熔断；超时预算层层递减（入口 10s→RPC 3s→DB 1s）。
- **复盘要点**：**超时是分布式系统的生命线**。没有显式超时的远程调用等于把线程生命周期交给最不可控的外部系统。

---

### 11.11 案例 10：ThreadLocal 泄漏导致 webapp 无法卸载

**来源**：Tomcat 官方 wiki《Memory Leaks - ClassLoader leaks》+ HeapDump 社区案例综合

- **症状**：热部署多次后元空间上涨不回落，最终 Metaspace OOM；每次 redeploy 都加重。
- **排查**：MAT Class Loader Explorer 显示多个旧 WebappClassLoader 未被回收 → Path to GC Roots：线程池里的常驻线程 ThreadLocalMap 持有旧加载器加载的对象 → 该线程是应用自建池，webapp 卸载时未 shutdown。
- **根因**：常驻线程（父容器级）持有 webapp 类加载器加载对象的强引用 → 加载器卸载条件永远不满足。ThreadLocal 在线程池场景下 set 后不 remove 就是泄漏。
- **修复**：应用停止钩子里 shutdown 自建线程池；Filter/Interceptor finally 里 remove ThreadLocal；升级 Tomcat 利用其卸载期清理告警定位。
- **复盘要点**：类加载器泄漏的排查口诀——**找到没死的线程，顺着 ThreadLocal 找到不该活的引用**。

---

### 11.12 复盘方法论：从个案到体系

| 维度 | 提炼动作 |
|------|---------|
| 止血 | 每个案例记录「当时怎么快速恢复的」，沉淀成一键预案 |
| 监控补位 | 问「哪个指标能让我下次提前 30 分钟知道」——队列深度/throttling/RSS 斜率都是这样补上的 |
| 代码防线 | 把根因翻译成编码规约检查项（超时必填/缓存三数字/异常路径 release） |
| 演练验证 | 修复后主动注入同类故障（混沌工程）验证防线有效 |
| 归档 | 案例编号入库，新人培训教材 = 本章节 |

---

### 11.13 考点速查表

| 考点 | 一句话要点 |
|------|-----------|
| 正则回溯案 | 嵌套量词+未覆盖字符=CPU 打满；独占模式根治 |
| 锁风暴案 | BLOCKED 集中同一地址→查持锁者栈顶；锁内禁 IO |
| 无界队列案 | FixedThreadPool+慢下游=慢性 OOM；有界队列是神经末梢 |
| 元空间泄漏案 | 动态类服务必答「旧 Loader 谁释放」；GC.class_loader_stats 一眼定位 |
| 137 案 | Xmx≤75% limit+封顶无界项+JDK≥8u191 三件套 |
| Netty 泄漏案 | 异常路径漏 release 最隐蔽；paranoid 档抓现行 |
| Humongous 案 | 周期性毛刺先对齐定时任务；G1 先查大对象 |
| 缓存膨胀案 | 上线前必须答出容量/大小/TTL 三个数字 |
| 假死传导案 | 无超时外呼+连接池=故障传导链；超时预算层层递减 |
| ThreadLocal 案 | 线程池 set 不 remove 即泄漏；顺 ThreadLocal 找钉住 Loader 的线程 |

---

[⬅️ 上一章](10-offheap-container.md) · [📖 返回目录](README.md) · [➡️ 下一章](12-jvm-flags.md)
