
[⬅️ 上一章](11-production-cases.md) · [📖 返回目录](README.md) · [➡️ 下一章](13-appendix-references.md)
# 12 · JVM 参数速查：常用参数、版本变化与危险黑名单

> **📌 30 秒速览**
> 1. 生产参数原则：**少即是多**——JVM 默认值经过大规模验证，只调你理解且有数据支撑的参数；每个参数改动都要有前后对比。
> 2. 容器基线四件套：`MaxRAMPercentage=75` + `MaxMetaspaceSize` 封顶 + `MaxDirectMemorySize` 封顶 + JDK ≥8u191。
> 3. 高危黑名单：`DisableExplicitGC`（堆外 OOM 帮凶）、`-Xmn` 与 G1 混用、老版本 CMS/PermGen 参数在新 JDK 上直接拒绝启动。
> 4. 排障三开关要会背：GC 日志（`-Xlog:gc*`）、OOM 自动 dump（HeapDumpOnOutOfMemoryError）、NMT（NativeMemoryTracking）——**平时就该开着**，出事再开就晚了。

---

### 12.1 必会参数速查

| 参数 | 含义 | 默认值 | 备注 |
|------|------|--------|------|
| `-Xms` / `-Xmx` | 初始/最大堆 | 物理内存 1/64 / 1/4 | 生产两者设相等避免伸缩抖动 |
| `-Xss` | 线程栈大小 | 1M（64位 Linux） | 高线程数服务可降 512k |
| `-XX:MetaspaceSize` | 元空间回收触发水位 | ~20.8M | 不是初始大小！启动慢可适当调大 |
| `-XX:MaxMetaspaceSize` | 元空间上限 | 无限 | **生产必封顶**，防 native 被吃光 |
| `-XX:MaxDirectMemorySize` | 直接内存上限 | ≈Xmx | 显式设置，别让它默认跟随堆 |
| `-XX:ReservedCodeCacheSize` | JIT 代码缓存 | 240M~512M(按版本) | 大量动态类生成时注意关闭分层后耗尽 |
| `-XX:MaxGCPauseMillis` | G1 目标停顿 | 200ms | 别设 <50ms，G1 会压缩吞吐换停顿 |
| `-XX:G1HeapRegionSize` | G1 Region 大小 | 堆/2048 取幂 (1~32M) | Humongous 频繁时调大 |
| `-XX:+HeapDumpOnOutOfMemoryError` | OOM 自动 dump | 关 | **生产必开**+配 HeapDumpPath |
| `-XX:+PrintGCDateStamps` 系(JDK8) / `-Xlog:gc*`(9+) | GC 日志 | 关(JDK8)/简(9+) | **生产必开**，落盘保留 |

---

### 12.2 GC 日志参数的版本分界（高频坑）

```bash
# JDK 8 写法(在 11+ 已移除,混用直接启动失败)
-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/var/log/gc.log
# JDK 9+ 统一写法(-Xlog 是统一日志框架)
-Xlog:gc*,safepoint:file=/var/log/gc.log:time,uptime,level,tags:filecount=5,filesize=50m
```

| 场景 | JDK8 | JDK9+ |
|------|------|-------|
| GC 详情 | PrintGCDetails | -Xlog:gc* |
| 时间戳 | PrintGCDateStamps | :time 标签 |
| 安全点 | PrintSafepointStatistics | -Xlog:safepoint |
| 类加载 | -verbose:class | -Xlog:class+load |

> 面试常考：为什么老参数在 JDK11 报 `Unrecognized VM option`？因为 JEP 158/JEP 271 重构了日志体系，旧 Print* 开关被移除而非废弃。迁移工具 `jcmd PID print_flags` 可对照当前生效值。

---

### 12.3 容器环境推荐基线（可直接抄）

```bash
# ===== 内存 =====
-XX:MaxRAMPercentage=75.0
-XX:InitialRAMPercentage=75.0
-XX:MaxMetaspaceSize=256m
-XX:MaxDirectMemorySize=512m
-Xss512k
# ===== GC(G1 默认即可满足大多数场景) =====
-XX:+UseG1GC
-XX:MaxGCPauseMillis=100            # 低延迟诉求再调,否则保持默认200
# ===== 排障(平时常开,开销极低) =====
-Xlog:gc*:file=/var/log/gc-%t.log:time,uptime,tags:filecount=5,filesize=50m
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/data/dump/
-XX:NativeMemoryTracking=summary
# ===== OOM 后自动重启配合(K8s) =====
-XX:+ExitOnOutOfMemoryError          # OOM 即退出让编排系统重建,避免僵尸状态
```

物理机大堆低延迟场景替换项：ZGC（JDK15+/推荐 17+）：`-XX:+UseZGC -XX:MaxRAMPercentage=75`，JDK21 用分代 ZGC（23 起为默认形态）。吞吐优先批处理：`-XX:+UseParallelGC`。

---

### 12.4 危险参数黑名单

| 参数 | 为什么危险 |
|------|-----------|
| `-XX:+DisableExplicitGC` | 让 DirectBuffer 分配失败的 System.gc() 兜底失效 → 堆外 OOM。用 `ExplicitGCInvokesConcurrent` 替代 |
| `-Xmn` / `-XX:NewRatio` 配 G1 | 显式新生代会禁用 G1 自适应暂停目标，等于自废武功 |
| `-XX:-UseCompressedOops`(64位) | 白白多耗内存，除非堆 >32G 且特殊调试 |
| PermGen/CMS 系参数在新 JDK | MaxPermSize 在 8 报警告、14 起 CMS 移除(JEP363) 直接拒绝启动 |
| `-XX:+AggressiveOpts` | 已废弃(JDK11+)，行为不可控 |
| 盲调 `-XX:MaxTenuringThreshold` | 对象晋升年龄机制复杂，乱调制造提前晋升或复制浪费 |
| `System.gc()` 调用点 | 不是参数但同罪：RMI/堆外分配外的显式调用应代码审查消灭 |

---

### 12.5 各 JDK 版本默认值关键变化（联网核实要点）

| 变化点 | 版本 | 说明 |
|--------|------|------|
| G1 成为默认 GC | JDK 9（JEP 248） | 8 及以前 server 默认 Parallel GC |
| CMS 废弃→移除 | JDK 9（JEP 291）→14（JEP 363） | UseConcMarkSweepGC 先警告后报错 |
| 偏向锁默认禁用→废弃 | JDK 15（JEP 374） | 「偏向锁提速」结论已过时 |
| ZGC 实验→正式 | JDK 11（JEP 333）→15（JEP 377） | 分代 ZGC 于 21 引入(JEP439)、23 成默认模式(JEP474) |
| Shenandoah 分代转正 | JDK 25（JEP 521） | 默认仍单代 |
| 容器感知 | 8u191+/10（JEP 298） | 老版本是容器事故之源 |
| RMI 的分布式 GC 改动、字符串去重等微调 | 各版本不一 | 以 `jcmd print_flags -all` 实测为准 |

> 版本事实均经 openjdk.org JEP 页面核实（2026-08），详见 13-附录。

---

### 12.6 常见误区

| 误区 | 正确认知 |
|------|---------|
| 参数越多越专业 | 每个非默认参数都是长期负债；默认值优先 |
| MetaspaceSize 是元空间初始大小 | 它是首次回收触发的水位线 |
| DisableExplicitGC 无副作用 | DirectBuffer 兜底回收被废，堆外 OOM 风险陡增 |
| G1 一定要配 Xmn 调新生代 | 相反：显式新生代破坏 G1 自适应模型 |
| GC 日志开销大不能开 | 异步落盘开销 <1%；「出事才开」等于没有黑匣子 |
| 新参数老 JDK 通吃 | 日志/GC 参数跨版本差异大，升级前先跑 `print_flags` 对比 |

---

### 12.7 面试题精选（含追问）

**Q1：生产环境你会配置哪些 JVM 参数？为什么？（追问：如果只允许三个呢？）**

答：分层回答：内存层（MaxRAMPercentage 或固定 Xmx=Xms、Meta/Direct 封顶）；GC 层（默认 G1 + 视 SLA 调 MaxGCPauseMillis）；排障层（GC 日志轮转、OOM dump、NMT summary、ExitOnOutOfMemoryError）。强调每个参数有监控依据。追问：只留三个我会选 HeapDumpOnOutOfMemoryError（事后能破案）、GC 日志（事中能看到趋势）、MaxMetaspaceSize 封顶（把无界的隐患变有限的问题）——都是「保命」而不是「提速」。

**Q2：-XX:+DisableExplicitGC 到底惹了什么祸？（追问：正确替代方案？）**

答：它禁止 System.gc() 生效，而 DirectByteBuffer 在额度不足时会主动调用 System.gc() 回收废弃 Buffer——被禁后这条兜底路径失效，堆外内存只能等自然 GC，高分配速率下很快 OOM: Direct buffer memory。追问：替代是 -XX:+ExplicitGCInvokesConcurrent，让显式 GC 走并发收集（G1/ZGC 下停顿可控），既保留堆外兜底又不引入长停顿；同时审查代码消灭无意义的 System.gc() 调用。

**Q3：为什么 JDK9 之后老的 GC 日志参数会启动失败？（追问：怎么平滑迁移？）**

答：JEP 158 引入统一 JVM 日志框架 -Xlog，JDK11 前后旧的 PrintGC* 系列被彻底移除（不是 deprecated），Unrecognized VM option 直接拒绝启动——这也是很多老应用升 11 的第一道坎。追问：迁移映射表：PrintGCDetails→-Xlog:gc*、PrintGCDateStamps→time 标签、PrintSafepointStatistics→-Xlog:safepoint、-verbose:class→-Xlog:class+load；先在测试环境用新参数跑压测比对输出完整性，再用启动脚本按 JDK 版本分支切换。

**Q4：如何评估一次 GC 参数调整是否有效？（追问：线上灰度怎么做？）**

答：定义指标先行：YGC/FGC 频率、P99/P999 停顿、吞吐比、分配速率、晋升速率。改前采基线 ≥1 周，改后同周期对比，用统计口径（分位数）而不是平均值。关注副作用：暂停目标过激进会牺牲吞吐，Region 调大会增加单次回收工作量。追问：灰度三板斧：①单实例金丝雀观察一周指标无回归；②按机房/集群逐步放量；③保留一键回滚配置（参数放配置中心热更，如 G1HeapRegionSize 等支持运行期调整的参数用 jcmd 管理）。

**Q5：容器里 JVM 显示的可用处理器数量和实际不符，会有什么后果？（追问：哪些行为受它影响？）**

答：availableProcessors() 决定了大量运行时决策：GC 并行线程数、JIT 编译线程数、ForkJoinPool.commonPool 并行度、Tomcat 默认线程公式等。老 JDK(≤8u131) 不感知 cgroup，在 64 核宿主机上这些全部按 64 配置，而容器 limit 可能只有 2 核——线程过多加剧 throttling，上下文切换暴涨。追问：修复路径：升级到感知版本；应急 -XX:ActiveProcessorCount 手动指定；验证 jcmd PID System.print_properties 或打印 Runtime.availableProcessors() 确认与 quota 一致。

---

### 12.8 考点速查表

| 考点 | 一句话要点 |
|------|-----------|
| 参数哲学 | 少即是多；默认优先；每次改动带前后数据 |
| 容器四件套 | RAMPercentage75 + Meta 封顶 + Direct 封顶 + JDK≥8u191 |
| 排障三开关 | gc* 日志 / HeapDumpOnOOM / NMT——平时常开当黑匣子 |
| 日志版本分界 | JDK8 Print* vs 9+ -Xlog 统一框架；混用直接启动失败 |
| 黑名单之首 | DisableExplicitGC 断堆外兜底；用 ExplicitGCInvokesConcurrent 替代 |
| G1 禁忌 | 别配 Xmn/NewRatio；别把 PauseMillis 压到 <50ms |
| MetaspaceSize 语义 | 回收触发水位线，不是初始容量 |
| 版本硬事实 | G1 默认=9；CMS 移除=14；偏向锁禁用=15；分代 ZGC 默认=23 |

---

[⬅️ 上一章](11-production-cases.md) · [📖 返回目录](README.md) · [➡️ 下一章](13-appendix-references.md)
