# CPU

## CPU Throttling

### 背景

最近公司在服务迁移上云（90% java），如何分配服务配额成为一个topic专项解决。

- v1: 从公司现有监控平台拉取JVM的cpu监控与启动参数，按照一定缩放作为用户画像向上取整
- 按照v1版本分配发现微服务长尾效应，许多边缘服务不到使用率不到1核，按照v1版本的画像进行分配默认1C6G（1.公司的资源配比为CPU:MEM=1:6左右，2.jvm堆大小默认4G，预留25%左右堆外）
- v2: 发现按照v1的配置带来了两个问题，因此v2按照k8s的监控提取画像，按下不表
  - Q1: JVM启动优化会使用更多的核，由于limit，Java服务的启动比物理机时代慢了1倍左右，对照监控发现与CPU Throttling指标强相关
  - Q2: 很多服务上云之后出现超时告警，对照监控发现与CPU Throttling指标强相关



### 概念

那么今天的主角CPU Throttling监控是如何计算的？开始面向搜索引擎debug。

#### 1. CFS调度器

>  Linux 在 2.6.23 之后开始引入 CFS 逐步替代 O1 调度器作为新的进程调度器，正如它名字所描述的，**CFS(Completely Fair Scheduler) 调度器[1]**追求的是对所有进程的全面公平，实际上它的做法就是在一个特定的调度周期内，保证所有待调度的进程都能被执行一遍，主要和当前已经占用的 CPU 时间经权重除权之后的值 (vruntime，见下面公式) 来决定本轮调度周期内所能占用的 CPU 时间，vruntime 越少，本轮能占用的 CPU 时间越多；总体而言，CFS 就是通过保证各个进程 vruntime 的大小尽量一致来达到公平调度的效果： 
>
> 进程的运行时间计算公式为: 进程运行时间  =  调度周期  *  进程权重  /  所有进程权重之和
>
> vruntime = 进程运行时间  * NICE_0_LOAD /  进程权重 = (调度周期  *  进程权重  /  所有进程总权重) * NICE_0_LOAD /  进程权重  =  调度周期  * NICE_0_LOAD /  所有进程总权重

除了CFS算法以外还有RT算法，本文不展开讨论，CFS通过下列参数控制行为：



 **强制上限的可调参数** 

> cpu.cfs_period_us
>
> 此参数可以设定重新分配 cgroup 可用 CPU 资源的时间间隔，单位为微秒（µs，这里以 “*`us`*” 表示）。如果一个 cgroup 中的任务在每 1 秒钟内有 0.2 秒的时间可存取一个单独的 CPU，则请将 `cpu.rt_runtime_us` 设定为 `2000000`，并将 `cpu.rt_period_us` 设定为 `1000000`。`cpu.cfs_quota_us` 参数的上限为 1 秒，下限为 1000 微秒。
>
> cpu.cfs_quota_us
>
> 此参数可以设定在某一阶段（由 `cpu.cfs_period_us` 规定）某个 cgroup 中所有任务可运行的时间总量，单位为微秒（µs，这里以 "*`us`*" 代表）。一旦 cgroup 中任务用完按配额分得的时间，它们就会被在此阶段的时间提醒限制流量，并在进入下阶段前禁止运行。如果 cgroup 中任务在每 1 秒内有 0.2 秒，可对单独 CPU 进行存取，请将 `cpu.cfs_quota_us` 设定为 `200000`，`cpu.cfs_period_us` 设定为 `1000000`。请注意，配额和时间段参数都根据 CPU 来操作。例如，如要让一个进程完全利用两个 CPU，请将 `cpu.cfs_quota_us` 设定为 `200000`，`cpu.cfs_period_us` 设定为 `100000`。
>
> 如将 `cpu.cfs_quota_us` 的值设定为 `-1`，这表示 cgroup 不需要遵循任何 CPU 时间限制。这也是每个 cgroup 的默认值（root cgroup 除外）。
>
> cpu.stat
>
> 此参数通过以下值报告 CPU 时间统计：
>
> - `nr_periods` — 经过的周期间隔数（如 `cpu.cfs_period_us` 中所述）。
> - `nr_throttled` — cgroup 中任务被节流的次数（即耗尽所有按配额分得的可用时间后，被禁止运行）。
> - `throttled_time` — cgroup 中任务被节流的时间总计（以纳秒为单位）。



 **相对比例的可调参数** 

>cpu.shares
>
>此参数用一个整数来设定 cgroup 中任务 CPU 可用时间的相对比例。例如： `cpu.shares`设定为 `100` 的任务，即便在两个 cgroup 中，也将获得相同的 CPU 时间；但是 `cpu.shares` 设定为 `200` 的任务与 `cpu.shares` 设定为 `100` 的任务相比，前者可使用的 CPU 时间是后者的两倍，即便它们在同一个 cgroup 中。`cpu.shares` 文件设定的值必须大于等于 `2`。
>
>
>
> 请注意：在多核系统中，CPU 时间比例是在所有 CPU 核中分配的。**即使在一个多核系统中，某个 cgroup 受限制而不能 100% 使用 CPU，它仍可以 100% 使用每个单独的 CPU 核**。请参考以下示例：如果 cgroup `A` 可使用 CPU 的 25%，cgroup `B` 可使用 CPU 的 75%，在四核系统中启动四个消耗大量 CPU 的进程（一个在 `A` 中，三个在 `B` 中）后，会得到以下 CPU 分配结果： 



#### 2. kernel配置

> 在 kernel 文件系统中，可以通过调整如下几个参数来改变 CFS 的一些行为：
>
> - /proc/sys/kernel/sched_min_granularity_ns，表示进程最少运行时间，防止频繁的切换，对于交互系统
> - /proc/sys/kernel/sched_nr_migrate，在多 CPU 情况下进行[负载均衡](https://cloud.tencent.com/product/clb?from=10680)时，一次最多移动多少个进程到另一个 CPU 上
> - /proc/sys/kernel/sched_wakeup_granularity_ns，表示进程被唤醒后至少应该运行的时间，这个数值越小，那么发生抢占的概率也就越高
> - /proc/sys/kernel/sched_latency_ns，表示一个运行队列所有进程运行一次的时间长度 (正常情况下的队列调度周期，P)
> - sched_nr_latency，这个参数是内核内部参数，无法直接设置，是通过 sched_latency_ns/sched_min_granularity_ns 这个公式计算出来的；在实际运行中，如果队列排队进程数  nr_running > sched_nr_latency，则调度周期就不是 sched_latency_ns，而是 P = sched_min_granularity_ns * nr_running，如果  nr_running <= sched_nr_latency，则  P = sched_latency_ns



#### 3. docker/k8s

docker的实现其实依赖于操作系统的两个feature：

1. control group
2. 联合文件系统

当kubernetes的一个POD定义了request/limit时，本质上反应在操作系统上是对应的cgroup配置

// TODO: 需要确认

-  **cpu.cfs_period_us** = 100ms，部分文章说明kubernetes在某个alpha版本中可以指定该配置，但查阅kubernetes文档发现kubelet可以通过某个参数调节
-  **cpu.cfs_quota_us**  = {limit} * 100
-  **cpu.shares**  = {request} * 1024 / 1000

 

### how it works

cpu利用率是均值，所以难以达到limit，但在更细的纬度上，进程对CPU的使用是不均匀的，因此会产生所谓的毛刺

下图展示了一次毛刺的产生过程：

令两个pod的limit值分别为50ms和20ms，request都为1000m

1. 因为两个pod的request值相同，share相等，所以在一个CFS调度周期里，两个pod都可以分别使用10ms中的5ms
2. 经过4个周期后，POD B达到了limit 20ms，此时所有的CPU资源都让给了POD A，在没有其他POD竞争的情况下，POD A独占 CFS调度周期中的全部10ms
3. 等到POD A也达到limit，此时即使有CPU资源，也不会分配给POD A 而会产生限流，限流时间累加在cpu.stat的`throttled_time` 记录中最终产生 CPU Throttling

![img](E:\github\acshana.github.io\static\kubernetes\cpu_period.png) 



### 解决方案

目前CPU限流的解决方案主要有两种

1. 增大limit，即超分，但需要注意集群整体的超分比例
2. linux 5.18 引入了CPU burst机制



### 参考文献

> [0]red_hat_enterprise_linux documentation https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/7/html/resource_management_guide/sec-cpu
>
> [1] K8S 中的 CPUThrottlingHigh 到底是个什么鬼？https://cloud.tencent.com/developer/article/1736729
>
> 



# MEMORY

