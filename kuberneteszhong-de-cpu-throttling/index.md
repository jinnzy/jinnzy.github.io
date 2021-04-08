# kubernetes中的cpu throttling


最近总是接到研发同学反馈线上应用响应慢或无响应及健康检测失败重启之类，查找发现是因为`cpu throttling` (cpu 节流/受限)。因为线上所有服务都是开启了cpu limit，会导致节流这一现象的发生。

![e196dd669f902c3df0374e1d6a19a323d31648c3.png](/img/e196dd669f902c3df0374e1d6a19a323d31648c3.png)![6fc05487c3f91da16d91c58d27e0bb09113e11e6.png](/img/6fc05487c3f91da16d91c58d27e0bb09113e11e6.png)

# k8s如何实现的资源划分

k8s使用`cgroup` 进行资源分配，`cgroup` 是具有层级结构的

```
/sys/fs/cgroup
├── blkio
├── cpu -> cpu,cpuacct
├── cpuacct -> cpu,cpuacct
├── cpu,cpuacct
│   ├── cgroup.clone_children
│   ├── cgroup.event_control
│   ├── cgroup.procs
│   ├── cgroup.sane_behavior
│   ├── cpuacct.stat
│   ├── cpuacct.usage
│   ├── cpuacct.usage_percpu
│   ├── cpu.cfs_period_us
│   ├── cpu.cfs_quota_us
│   ├── cpu.rt_period_us
│   ├── cpu.rt_runtime_us
│   ├── cpu.shares
│   ├── cpu.stat
│   ├── kubepods.slice
│   ├── kube.slice
│   ├── notify_on_release
│   ├── release_agent
│   ├── system.slice
│   ├── tasks
│   └── user.slice
├── cpuset
│   ├── cgroup.clone_children
│   ├── cgroup.event_control
│   ├── cgroup.procs
│   ├── cgroup.sane_behavior
│   ├── cpuset.cpu_exclusive
│   ├── cpuset.cpus
│   ├── cpuset.effective_cpus
│   ├── cpuset.effective_mems
│   ├── cpuset.mem_exclusive
│   ├── cpuset.mem_hardwall
│   ├── cpuset.memory_migrate
│   ├── cpuset.memory_pressure
│   ├── cpuset.memory_pressure_enabled
│   ├── cpuset.memory_spread_page
│   ├── cpuset.memory_spread_slab
│   ├── cpuset.mems
│   ├── cpuset.sched_load_balance
│   ├── cpuset.sched_relax_domain_level
│   ├── kubepods.slice
│   ├── kube.slice
│   ├── notify_on_release
│   ├── release_agent
│   ├── system.slice
│   └── tasks
├── devices
├── freezer
├── hugetlb
├── memory
│   ├── cgroup.clone_children
│   ├── cgroup.event_control
│   ├── cgroup.procs
│   ├── cgroup.sane_behavior
│   ├── kubepods.slice
│   ├── kube.slice
│   ├── memory.failcnt
│   ├── memory.force_empty
│   ├── memory.kmem.failcnt
│   ├── memory.kmem.limit_in_bytes
│   ├── memory.kmem.max_usage_in_bytes
│   ├── memory.kmem.slabinfo
│   ├── memory.kmem.tcp.failcnt
│   ├── memory.kmem.tcp.limit_in_bytes
│   ├── memory.kmem.tcp.max_usage_in_bytes
│   ├── memory.kmem.tcp.usage_in_bytes
│   ├── memory.kmem.usage_in_bytes
│   ├── memory.limit_in_bytes
│   ├── memory.max_usage_in_bytes
│   ├── memory.memsw.failcnt
│   ├── memory.memsw.limit_in_bytes
│   ├── memory.memsw.max_usage_in_bytes
│   ├── memory.memsw.usage_in_bytes
│   ├── memory.move_charge_at_immigrate
│   ├── memory.numa_stat
│   ├── memory.oom_control
│   ├── memory.pressure_level
│   ├── memory.soft_limit_in_bytes
│   ├── memory.stat
│   ├── memory.swappiness
│   ├── memory.usage_in_bytes
│   ├── memory.use_hierarchy
│   ├── notify_on_release
│   ├── release_agent
│   ├── system.slice
│   ├── tasks
│   └── user.slice
├── net_cls -> net_cls,net_prio
├── net_cls,net_prio
├── net_prio -> net_cls,net_prio
├── perf_event
├── pids
│   ├── cgroup.clone_children
│   ├── cgroup.event_control
│   ├── cgroup.procs
│   ├── cgroup.sane_behavior
│   ├── kubepods.slice
│   ├── kube.slice
│   ├── notify_on_release
│   ├── pids.current
│   ├── release_agent
│   ├── system.slice
│   ├── tasks
│   └── user.slice
└── systemd
    ├── cgroup.clone_children
    ├── cgroup.event_control
    ├── cgroup.procs
    ├── cgroup.sane_behavior
    ├── kubepods.slice
    ├── kube.slice
    ├── notify_on_release
    ├── release_agent
    ├── system.slice
    ├── tasks
    └── user.slice
```

## cpu资源

### request

cpu request 是通过`cpu,cpuacct/kubepods.slice/pod<UID>/xxx/cpu.shares` 来实现的

![17108e373490d32e0607d4a8b344f5b866de30ee.png](/img/17108e373490d32e0607d4a8b344f5b866de30ee.png)

假设这是一个4c的节点，根cgroup为`4096` 个cpu，子目录cpu.shares的总数却超过了4096,在这种情况下linux cfs会按照cpu.shares值的比例来分配cpu,如：system.slice user.slice获得680个cpu(4096的16.6)，kubepod获得2736个。 如果在空闲情况下前两个cgroup不会使用分配的cpu资源，调度程序避免为使用的cpu浪费，将会把这部分释放到全局资源中，以便分配给需要更多cpu的cgroup

当你指定了某个容器的 CPU request 值为 x millicores 时，kubernetes 会为这个 container 所在的 cgroup 的 cpu.shares 的值指定为 x * 1024 / 1000。即：cpu.shares = (cpu in millicores * 1024) / 1000。 cpu 1核等于1000 millicores

如果资源空闲时，容器可以使用limit上限的cpu,但是当cpu资源不足时会保证容器有request cpu的资源可用

例子：服务的资源设置如下：

```
resources:
          limits:
            cpu: "1"
            memory: 1Gi
          requests:
            cpu: 500m
            memory: 512Mi
```

登陆节点获取pid，查看cpu.shares的值为512，结果为预期的值

```bash
docker ps |grep fs-mankeep-task
k8s_POD_fs-mankeep-task-656fd776b4-vqznd_fstest_12debdb4-29a5-4fb2-b971-3a8144909a67_0

cat /sys/fs/cgroup/cpu,cpuacct/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod12debdb4_29a5_4fb2_b971_3a8144909a67.slice/cpu.shares 
512
```

### limit

kubernetes通过 cpu cgroup中的`cpu.cfs_period_us cpu.cfs_quota_us`两个配置实现的，会保证使用的cpu不会超过`cfs_quota_us/cfs_period_us` 的值。

与cpu.share不同的是，quota是基于时间段来限制

`cpu.cfs_period_us` 周期时间

`cpu.cfs_quota_us` 时间份额，表示在1个period周期允许使用的最大额度

```
cpu.cfs_period_us = 100000 (i.e. 100ms)
cpu.cfs_quota_us = quota = (cpu in millicores * 100000) / 1000
```

- memory 内存资源
- kubelet默认使用各个资源下的`kubepods.slice`目录，和kubelet配置有关
- kubelet会在`kubepods.slice` 下面为每个pod创建一个`pod<pod.UID>` 的cgroup值

    cpu request值，如果容器未指定limit值，则只设置cpu.shares，如果pod在节点资源空闲时可以无限制使用cpu资源，但是资源不足时至少会保证可以使用到cpu.shares的值

```
pod<UID>/cpu.shares
```

    cpu limit值

```
pod<UID>/cpu.cfs_quota_us
```

    memory limit值

```
pod<UID>/memory.limit_in_bytes
```

    # 参考文章

    [https://zhuanlan.zhihu.com/p/38359775](https://zhuanlan.zhihu.com/p/38359775)

    [https://medium.com/omio-engineering/cpu-limits-and-aggressive-throttling-in-kubernetes-c5b20bd8a718](https://medium.com/omio-engineering/cpu-limits-and-aggressive-throttling-in-kubernetes-c5b20bd8a718)

