
[⬅️ 上一章](12-jvm-flags.md) · [📖 返回目录](README.md)
# 13 · 附录：版本事实与参考资料

> **本章定位**：手册中所有版本号、JEP 编号、工具版本的核实记录与来源索引。事实采集时间 **2026-08-25**，由联网调研子代理逐条核对原始出处（OpenJDK JEP 页面、GitHub Releases API、官方文档）。

---

### 13.1 JDK 发布节奏

- 6 个月一个 Feature Release 自 **JDK 9（2017-09）** 起，Mark Reinhold 提案生效。来源：mreinhold.org/blog/forward-faster（经 Wikipedia Java version history 页交叉引用）。
- LTS 版本：**8 / 11 / 17 / 21 / 25**。
- Oracle 对个人/开发用途无限期提供 JDK 8 免费公开更新；商业 Premier Support 延续至 2030+。第三方发行版（Eclipse Temurin 等）免费支持至少到 2026 年以后。

### 13.2 GC 演进大事记（JEP 均已逐一核对 openjdk.org）

| 收集器 | 引入 | 关键里程碑 |
|--------|------|-----------|
| Parallel GC | JDK 1.4 时代逐步成型 | JDK 8 默认（server class）；JEP 366(14) 废弃 ParallelScavenge+SerialOld 组合 |
| CMS | JDK 1.4 | JEP 291(**9**) 废弃 → JEP 363(**14**) 移除源码 |
| G1 | JDK 7u4 官方支持 | JEP 248(**9**) 成为 server 默认；JEP 423(22) Region Pinning；JEP 475(24) Late Barrier Expansion |
| ZGC | JEP 333(**11**, Experimental, Linux/x64) | JEP 377(**15**) 转正；JEP 439(**21**) 分代 ZGC；JEP 474(**23**) 分代为默认模式；JEP 490(**24**) 删除非分代形态 |
| Shenandoah | JEP 189(**12**, Experimental) | 不在 Oracle JDK 构建；JEP 404(24) 分代实验；JEP 521(**25**) 分代转正但默认仍单代 |
| Epsilon | JEP 318(**11**, Experimental) | No-Op GC，测试用 |

**各版本默认 GC**：8 = Parallel；9~25 = G1（ZGC/Shenandoah 从不自动成为默认；「分代 ZGC 默认」仅指启用 UseZGC 后的内部形态，JDK23 起）。

### 13.3 类加载相关版本事实

| 事件 | 版本 | 出处 |
|------|------|------|
| PermGen 移除，元数据入 Metaspace | JDK 8（JEP 122） | openjdk.org/jeps/122 |
| 字符串常量池移入堆 | JDK 7 | HotSpot 变更说明 |
| 偏向锁默认禁用并废弃 | JDK 15（JEP 374） | openjdk.org/jeps/374 |

### 13.4 容器感知事实

- JDK 10 引入容器感知（JEP 298 + 相关修复集），`UseContainerSupport` 默认开启；
- Backport 至 **8u191**：从此 8u191+ 在容器内正确读取 cgroup CPU/内存限制；
- ≤8u131 版本完全不感知——容器化事故高发版本线。

### 13.5 工具版本现状（GitHub Releases API 核实，2026-08）

| 工具 | 最新版本 | 要点 |
|------|---------|------|
| Arthas (alibaba/arthas) | arthas-all-**4.3.4**（2026-08-13 发布），Apache 2.0 | watch/trace/tt/profiler/heapdump/vmtool/thread 等核心命令齐全；安装 `curl -O https://arthas.aliyun.com/arthas-boot.jar` |
| async-profiler (jvm-profiling-tools) | **v4.5**（2026-07-20） | 启动器已更名 **asprof**（旧 profiler.sh 弃用）：`asprof -d 30 -f flamegraph.html <PID>`；输出支持 collapsed/flamegraph/tree/text/jfr |
| Eclipse MAT | 社区版持续更新 | Dominator tree/Duplicate Classes/Class Loader Explorer 为排障三宝 |
| VisualVM | visualvm.github.io 持续发布 | GitHub 发行渠道 |
| JFR/JMC | 随 JDK 内置；JMC 独立发版 | 生产常驻推荐 default profile |

### 13.6 案例与资料来源索引

| 资料 | 用途 | 章节 |
|------|------|------|
| 陈树义《一个正则表达式引发的血案》(zhuanlan.zhihu.com/p/38229530) | ReDoS 案例 | 07/11 |
| 美团技术团队博客 tech.meituan.com（G1 问题、线程池实践、超时治理等系列） | GC/案例素材 | 06/08/11 |
| HeapDump 性能社区 heapdump.cn（MAT/Metaspace/泄漏系列） | 内存类案例 | 04/09/11 |
| 阿里《Java 开发手册》（嵩山版及后续） | 线程池规约、编码防线 | 08/12 |
| Tomcat wiki《Memory Leaks - ClassLoader leaks》 | 类加载器泄漏 | 09/11 |
| OpenJDK JEP 索引 openjdk.org/jeps/* | 全部版本事实 | 05/12/13 |
| Oracle《Java SE Support Roadmap》 | 支持周期 | 13 |

> 核实方法说明：版本号与发布日期以 **api.github.com/repos/*/releases/latest** 与 openjdk.org/jeps/<N> 的 Release 字段为准；文档类内容经官方站点直接抓取或 r.jina.ai 代理阅读。本附录不收录任何未经证实的数字与结论。

---

### 13.7 考点速查表

| 考点 | 一句话要点 |
|------|-----------|
| 节奏与 LTS | 2017-09 起半年一发；LTS=8/11/17/21/25 |
| 默认 GC 分界 | 8=Parallel，9+=G1（JEP 248） |
| CMS 生卒 | 1.4 诞生 → 9 废弃(JEP291) → 14 移除(JEP363) |
| ZGC 三级跳 | 11 实验→15 正式→23 分代默认(JEP333/377/439/474) |
| 偏向锁终点 | JDK15 默认禁用废弃（JEP374），旧结论勿再用 |
| PermGen 终点 | JDK8 移除（JEP122），字符串池 7 已入堆 |
| 容器感知分界 | 10 原生(JEP298)，backport 到 8u191 |
| asprof 更名 | async-profiler v4.x 启动器叫 asprof，profiler.sh 已弃用 |

---

[⬅️ 上一章](12-jvm-flags.md) · [📖 返回目录](README.md)
