---
title: GC
date: 2020-11-19 08:31:57
tags: java
categories: java
cover: /img/topimg/202105161049.png
---

## GC 算法

## 重点关注的GC case
1、**System.gc()**： 手动触发GC操作。

2、**CMS**： CMS GC 在执行过程中的一些动作，重点关注 `CMS Initial Mark` 和 `CMS Final Remark` 两个 STW 阶段

3、**Promotion Failure**： Old 区没有足够的空间分配给 Young 区晋升的对象（即使总可用内存足够大）

4、**Concurrent Mode Failure**： CMS GC 运行期间，Old 区预留的空间不足以分配给新的对象，此时收集器会发生退化，严重影响 GC 性能，下面的一个案例即为这种场景

5、**GCLocker Initiated GC**： 如果线程执行在 JNI 临界区时，刚好需要进行 GC，此时 `GC Locker` 将会阻止 GC 的发生，同时阻止其他线程进入 JNI 临界区，直到最后一个线程退出临界区时触发一次 GC

## GC 问题分类：
**Unexpected GC**： 意外发生的 `GC`，实际上不需要发生，我们可以通过一些手段去避免
* **Space Shock**： 空间震荡问题，参见“场景一：动态扩容引起的空间震荡”
    **现象**：服务刚刚启动时 GC 次数较多，最大空间剩余很多但是依然发生 GC，一般为 `Allocation Failure`
    
    **原因**： JVM 的参数中 `-Xms` 和 `-Xmx` 设置的不一致，在初始化时只会初始 `-Xms` 大小的空间存储信息，每当空间不够用时再向操作系统申请，这样的话必然要进行一次 GC
    
    **定位**： 观察 CMS GC 触发时间点 Old/MetaSpace 区的 committed 占比是不是一个固定的值，或者观察内存使用率
    
    **策略**：尽量将成对出现的空间大小配置参数设置成固定的
    
    * **Explicit GC**： 显示执行 GC 问题，参见“场景二：显式 GC 的去与留”
    **现象**：手动调用了 `System.gc`

    **原因**： `System.gc` 会引发一次 STW 的 Full GC，对整个堆做收集，保留 `System.gc`，会有频繁调用的风险；去掉 `System.gc`，会发生内存泄漏（主要是堆外内存）
        
    **策略**：`-XX:+ExplicitGCInvokesConcurrent` 和 `-XX:+ExplicitGCInvokesConcurrentAndUnloadsClasses` 参数来将 `System.gc` 的触发类型从 Foreground 改为 Background（`-XX:+UseAdaptiveSizePolicyWithSystemGC`）
    
**Partial GC**： 部分收集操作的 GC，只对某些分代/分区进行回收
* **Young GC**： 分代收集里面的 Young 区收集动作，也可以叫做 Minor GC
    * **ParNew**： Young GC 频繁，参见“场景四：过早晋升”
        **现象**：分配速率接近于晋升速率，对象晋升年龄较小（`-XX:MaxTenuringThreshold`）；Full GC 比较频繁，且经历过一次 GC 之后 Old 区的变化比例非常大
            
        **原因**：Young/Eden 区过小；分配速率过大
                        
        **策略**：增大 Young 区，一般情况下 `Old` 的大小应当为活跃对象的 2~3 倍左右，考虑到浮动垃圾问题最好在 3 倍左右，剩下的都可以分给 Young 区
    
* **Old GC**： 分代收集里面的 Old 区收集动作，也可以叫做 Major GC，有些也会叫做 Full GC，但其实这种叫法是不规范的，在 CMS 发生 Foreground GC 时才是 Full GC，CMSScavengeBeforeRemark 参数也只是在 Remark 前触发一次 Young GC。
    * **CMS**： Old GC 频繁，参见“场景五：CMS Old GC 频繁”
        **现象**：Old 区频繁的做 `CMS GC，但是每次耗时不是特别长，整体最大 STW 也在可接受范围内，但由于 GC 太频繁导致吞吐下降比较多
                                                    
        **策略**：内存 Dump;分析 Top Component;分析 Unreachable
            
    * **CMS**： `Old GC` 不频繁但单次耗时大，参见“场景六：单次 `CMS Old GC` 耗时长”
        **现象**：CMS GC 单次 STW 最大超过 1000ms，不会频繁发生
              
        **原因**：CMS 在回收的过程中，STW 的阶段主要是 `Init Mark` 和 `Final Remark` 这两个阶段，也是导致 CMS Old GC 最多的原因
        
        **策略**：大部分问题都出在 `Final Remark` 过程;观察详细 GC 日志，找到出问题时 `Final Remark` 日志

**Full GC**： 全量收集的 `GC`，对整个堆进行回收，`STW` 时间会比较长，一旦发生，影响较大，也可以叫做 `Major GC`，参见“场景七：内存碎片&收集器退化”
    **现象**：并发的 CMS GC 算法，退化为 Foreground 单线程串行 GC 模式，STW 时间超长，有时会长达十几秒
    **原因**：晋升失败（Promotion Failed）；显式 GC；增量收集担保失败；并发模式失败（Concurrent Mode Failure）
    **策略**：
    
    内存碎片： 通过配置 `-XX:UseCMSCompactAtFullCollection=true`，`-XX: CMSFullGCsBeforeCompaction=n` 来控制多少次 Full GC 后进行一次压缩
    
    增量收集： 降低触发 CMS GC 的阈值，即参数 `-XX:CMSInitiatingOccupancyFraction` 的值，让 CMS GC 尽早执行，以保证有足够的连续空间，也减少 Old 区空间的使用大小，另外需要使用 `-XX:+UseCMSInitiatingOccupancyOnly` 来配合使用
    
    浮动垃圾： 视情况控制每次晋升对象的大小，或者缩短每次 CMS GC 的时间，必要时可调节 NewRatio 的值。另外就是使用 -XX:+CMSScavengeBeforeRemark 在过程中提前触发一次 Young GC，防止后续晋升过多对象。

**MetaSpace**： 元空间回收引发问题，参见“场景三：`MetaSpace` 区 `OOM`”
    **现象**：`JVM` 在启动后或者某个时间点开始，`MetaSpace` 的已使用大小在持续增长，同时每次 `GC` 也无法释放，调大 `MetaSpace` 空间也无法彻底解决。
    **原因**：空间不够（`-XX:MetaSpaceSize`， `-XX:MaxMetaSpaceSize`）
    **策略**：观察详细的类加载和卸载信息
    

**Direct Memory**： 直接内存（也可以称作为堆外内存）回收引发问题，参见“场景八：堆外内存 `OOM`”
    **现象**：内存使用率不断上升，甚至开始使用 SWAP 内存，通过 top 命令发现 Java 进程的 RES 甚至超过了 `-Xmx` 的大小
    **原因**：代码中有通过 JNI 调用 Native Code 申请的内存没有释放；通过 `UnSafe#allocateMemory`，`ByteBuffer#allocateDirect` 主动申请了堆外内存而没有释放
    **策略**：JVM 使用 `-XX:MaxDirectMemorySize=size` 参数来控制可申请的堆外内存的最大值
**JNI**： 本地 Native 方法引发问题，参见“场景九：JNI 引发的 GC 问题”    
    **现象**：在 GC 日志中，出现 GC Cause 为 GCLocker Initiated GC
    **策略**：添加 -XX+PrintJNIGCStalls 参数，可以打印出发生 JNI 调用时的线程，进一步分析，找到引发问题的 JNI 调用
    

## 分析工具
### 火焰图

[如何读懂火焰图](http://www.ruanyifeng.com/blog/2017/09/flame-graph.html)

工具：JProfiler
```shell script
$ sudo perf record -F 99 -p [进程id] -g -- sleep 30
$ perf script | ./stackcollapse-perf.pl > out.perf-folded
## 将 out.perf-folded 下载至本地

git clone https://github.com/brendangregg/FlameGraph
cd FlameGraph
./flamegraph.pl out.perf-folded > perf-kernel.svg

## 查看调用栈百分比
$ sudo perf report -n --stdio

```
[uber jvm-profiler 在**spark**或**java**上的应用](https://github.com/uber-common/jvm-profiler)
> http://www.brendangregg.com/FlameGraphs/cpuflamegraphs.html

###

## 参考
> https://tech.meituan.com/2020/11/12/java-9-cms-gc.html