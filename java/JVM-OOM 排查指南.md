# Java JVM OOM 问题排查指南

## 一、OOM 常见类型

```
1. Java heap space          - 堆内存不足
2. Metaspace / PermGen      - 元空间/永久代不足
3. GC overhead limit exceeded - GC 时间过长
4. OutOfMemoryError: Direct buffer memory - 直接缓冲区不足
5. OutOfMemoryError: StackOverflowError - 栈溢出
6. OutOfMemoryError: Unable to create new native thread - 无法创建线程
```

---

## 二、命令行工具排查

### 1. 查看 JVM 参数和内存配置

```bash
# 查看 JVM 启动参数
jps -v

# 查看堆内存配置
java -XX:+PrintFlagsFinal -version | grep -i heap

# 查看当前堆使用情况（JDK 8）
jstat -gc <pid> 1000 10  # 每秒打印一次，共 10 次
```

**jstat 输出解读**：
```
S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     YGC     YGCT    FGC    FGCT
# 年轻代        老年代         元空间   GC 次数   GC 总时间
```

### 2. 生成堆转储文件（Heap Dump）

```bash
# 方式 1: OOM 时自动生成（添加到启动参数）
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/path/to/dump.hprof

# 方式 2: 手动生成
jmap -dump:format=b,file=heap.hprof <pid>

# 方式 3: 只导出存活对象
jmap -dump:live,format=b,file=heap_live.hprof <pid>
```

### 3. 分析堆转储文件

```bash
# 查看直方图（对象数量统计）
jmap -histo <pid> | head -50

# 查看直方图（含存活对象）
jmap -histo:live <pid> | head -50

# 输出示例
num     #instances         #bytes  class name
----------------------------------------------
   1:        500000      20000000  java.lang.String
   2:        300000      15000000  [C
   3:        200000      10000000  java.util.HashMap$Node
```

### 4. 线程分析

```bash
# 生成线程转储
jstack <pid> > thread_dump.txt

# 查看线程数
ps -eLo pid,tid,class,psr,comm | grep <pid> | wc -l

# 查看占用 CPU 高的线程
top -Hp <pid>
# 然后 jstack 查找对应线程
printf "%x\n" <tid>  # 线程 ID 转 16 进制
```

### 5. 实时监控工具

```bash
# jstat 实时监控 GC
jstat -gcutil <pid> 1000

# jcmd 查看堆信息
jcmd <pid> GC.heap_info

# jcmd 触发 GC
jcmd <pid> GC.run

# jcmd 生成堆转储
jcmd <pid> GC.heap_dump /path/to/heap.hprof
```

---

## 三、图形化工具排查

### 1. VisualVM（推荐）

**连接方式**：
```bash
# 本地应用
直接显示在应用程序列表

# 远程应用
# 服务端添加 JVM 参数
-Dcom.sun.management.jmxremote
-Dcom.sun.management.jmxremote.port=9999
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.ssl=false
```

**功能**：
- 实时内存监控
- 线程分析
- 堆转储分析
- CPU 采样

### 2. JVisualVM（JDK 自带）

```bash
# 启动
jvisualvm

# 功能
- 监视堆内存使用
- 分析 GC 情况
- 线程 dump 分析
- 堆 dump 分析
```

### 3. Eclipse MAT（Memory Analyzer Tool）

**分析步骤**：
1. 打开 heap dump 文件
2. 查看 **Histogram**（对象直方图）
3. 查看 **Dominator Tree**（支配树）
4. 使用 **Path to GC Roots** 查找引用链
5. 查看 **Leak Suspects Report**（内存泄漏报告）

**关键功能**：
```
- Leak Suspects: 自动分析内存泄漏嫌疑
- Dominator Tree: 按保留内存排序
- OOM Analysis: OOM 分析报告
```

### 4. JMC（Java Mission Control）

```bash
# 启动
jmc

# 功能
- JDK Flight Recorder (JFR) 录制
- 内存泄漏检测
- GC 分析
- 线程分析
```

### 5. GCEasy（在线分析）

**网址**：https://gceasy.io/

**使用**：
1. 添加 JVM 参数输出 GC 日志
```bash
-Xloggc:/path/to/gc.log
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
```
2. 上传 gc.log 到网站
3. 查看分析报告

---

## 四、排查流程

### 标准排查步骤

```
┌─────────────────────────────────────────┐
│  1. 确认 OOM 类型                        │
│     查看错误堆栈信息                     │
└─────────────┬───────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────┐
│  2. 获取堆转储文件                       │
│     jmap -dump 或自动 dump               │
└─────────────┬───────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────┐
│  3. 分析堆转储                           │
│     MAT / VisualVM 分析                  │
│     找出占用内存最大的对象               │
└─────────────┬───────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────┐
│  4. 定位代码                             │
│     通过 GC Roots 追踪引用链             │
│     找到内存泄漏的代码位置              │
└─────────────┬───────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────┐
│  5. 验证修复                             │
│     压测验证 + 监控                      │
└─────────────────────────────────────────┘
```

### 快速排查命令组合

```bash
# 1. 找到 Java 进程
jps -l

# 2. 查看内存概况
jstat -gc <pid> 1000 5

# 3. 查看对象分布
jmap -histo:live <pid> | head -30

# 4. 生成堆转储
jmap -dump:format=b,file=/tmp/heap.hprof <pid>

# 5. 生成线程转储
jstack <pid> > /tmp/thread.txt

# 6. 使用 MAT 分析 heap.hprof
```

---

## 五、常见 OOM 场景及解决

### 1. Java heap space

**原因**：
- 内存泄漏（静态集合、未关闭资源）
- 内存不足（数据量过大）
- 对象过大

**解决**：
```bash
# 临时：增大堆内存
-Xms4g -Xmx4g

# 根本：修复内存泄漏
# 检查静态 Map/List
# 检查未关闭的连接
# 检查缓存无限制增长
```

### 2. Metaspace

**原因**：
- 加载类过多（动态代理、Groovy 脚本）
- 元空间设置过小

**解决**：
```bash
# 增大元空间
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=512m

# 检查动态生成类的代码
```

### 3. GC overhead limit exceeded

**原因**：
- GC 时间超过 98%
- 回收内存小于 2%

**解决**：
```bash
# 禁用此检查
-XX:-UseGCOverheadLimit

# 根本解决：增大堆或修复泄漏
-Xmx4g
```

### 4. Unable to create new native thread

**原因**：
- 线程数过多
- 系统线程限制

**解决**：
```bash
# 查看线程限制
ulimit -u

# 减少线程池大小
# 检查线程泄漏
jstack <pid> | grep "java.lang.Thread" | wc -l
```

---

## 六、预防建议

```yaml
启动参数建议:
  堆内存：-Xms4g -Xmx4g  # 设成一样避免震荡
  元空间：-XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m
  GC 日志：-Xloggc:/var/log/gc.log -XX:+PrintGCDetails
  OOM Dump: -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dump/
  GC 算法：-XX:+UseG1GC  # JDK 8 推荐 G1

监控建议:
  - 部署 Prometheus + Grafana 监控 JVM
  - 配置 OOM 告警
  - 定期分析 GC 日志
  - 压测时监控内存使用
```

---

## 七、实战案例

### 案例：内存泄漏排查

```bash
# 1. 发现 OOM 告警
# Error: Java heap space

# 2. 登录服务器
jps -l  # 找到进程 ID 12345

# 3. 查看堆情况
jstat -gc 12345 1000 5
# 发现老年代持续增长，GC 后不下降

# 4. 生成堆转储
jmap -dump:format=b,file=/tmp/oom.hprof 12345

# 5. 本地用 MAT 分析
# 打开 oom.hprof
# 查看 Leak Suspects Report
# 发现静态 HashMap 持有大量对象引用

# 6. 定位代码
# 找到静态 Map 的使用位置
# 修复：添加过期机制或使用 WeakHashMap

# 7. 验证
# 重新部署，压测观察内存稳定
```
