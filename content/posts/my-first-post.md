---
author: 1scream
title: "kube-proxy消耗大量CPU案例分析"
date: 2023-02-19T15:16:25+08:00
description: "在Linux环境下调试一个GO程序的流程"
categories:
- k8s
- golang
---

## introduce
此案例来自于云原生训练营孟凡杰老师的案例，想着记录一下老师的分析过程和思路，所以写下了这篇博客。

这里进入正题，如标题所写kube-proxy进程在机器中占用了10个CPU，kube-proxy正常情况下是不可能占用如此大量的CPU资源的，很有可能是bug，抱着怀疑的态度进行分析。

首先进入问题机器，使用[top](https://wangchujiang.com/linux-command/c/top.html)命令查看进程占用资源的情况，这时候可以看到kube-proxy占用了大量CPU的资源，下一步对kube-proxy的进程运行系统分析工具，使用[perf](https://www.cnblogs.com/arnoldlu/p/6241297.html)命令查看返回结果进行分析。

```bash
###top 显示了系统总体的 CPU 和内存使用情况,以及各个进程的资源使用情况。
top

###使用perf top -p <pid> Profile events on existing Process ID (comma sperated list).
###仅分析目标进程及其创建的线程。
perf top -p <pid>
OverHead Shared      Object  Symbol
26.48%   kube-proxy  [.]     runtime.gcDrain
13.86%   kube-proxy  [.]     runtime.greyObject
10.71%   kube-proxy  [.]     runtime.(*lfstack).pop
10.04%   kube-proxy  [.]     runtime.scanobject
```

通过perf的命令输出我们可以看到kube-proxy的占用情况，超过60%的占用都是发生在GC的过程中，这时候大家应该都会想到面试中经典的问题，小对象分配过多会造成GC的压力问题，分配小对象的时候根据内存分配器的分配策略会先从本线程的线程缓存中对应规格的mspan进行分配，如果当前线程的mcache内存空间不足了，那么就会向上一级的mcentral或者mheap申请新的mspan，但是这时如果mspan满了的话，就可能会触发垃圾回收。如果系统进行GC进度赶不上对象分配的速度，还会触发协助标记应用本身会适当协助GC进行标记，防止应用因为分配过快发生OOM，这一系列的问题都是可能导致CPU占用资源过高的原因。

通过golang自带的性能分析工具进行分析，kubernetes内置pprof的API可以直接ping链接。

```bash
curl 127.0.0.1:10249/debug/pprof/heap?debug=2
```

通过分析结果我们可以定位具体函数或者方法来debug。Get LocalAddress这个函数调用创建了30多万个对象，占用了大量的内存。如此大量的对象被创建，显然会导致kube-proxy进行GC，占用大量CPU资源。

最后对照GetLocalAddress的代码的实现，发现该函数的主要目的是获取本机的IP地址，获取的方法是同ip route命令获得当前节点所有local路由新兵转换成go struct并过滤掉ipvs0网口上的路由信息。这里因为开启了ipvs0模式，部署的集群有大量的service，每个service都有一个ip地址。

因为集群规模比较打，ip route show table local type proto kernel命令会返回大概5000多条记录，因此每次函数调用都会有数万个对象被生成，而kube-proxy在处理每一个服务的时候都会调用该方法，因为集群有数千个服务，因此，kube-proxy在反复调用该函数创建大量临时对象。

## Reference:

[https://github.com/kubernetes/kubernetes/pull/79444](https://github.com/kubernetes/kubernetes/pull/79444)

[https://www.youtube.com/watch?v=6A3MCycGF7g&list=PLOJ_KEizytL9gXQi8bxLWWGp4-MMJ1bLT&index=4](https://www.youtube.com/watch?v=6A3MCycGF7g&list=PLOJ_KEizytL9gXQi8bxLWWGp4-MMJ1bLT&index=4)