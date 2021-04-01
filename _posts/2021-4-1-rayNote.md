---
title: "AI, System: Ray(Presentation Note)"
categories:
  - AI
  - System
---

Project Ray：

* 应对深度强化学习的计算框架（？似乎不太成功）
* **“高性能 分布式 Python RPC 计算框架”**

和Spark(RDD)不同，Ray的调度核心是Tasks。

![Structure of Ray]({{ site.url }}{{ site.baseurl }}/assets/images/ray_struct.png){: .align-center}

1.0版本之后的架构可以简单地表示为：

![New Structure of Ray]({{ site.url }}{{ site.baseurl }}/assets/images/new_struct.png){: .align-center}

* Driver：应用主程序，目前（1.0架构）driver必须和集群内的一个节点绑定
* Worker：计算进程，默认是一个逻辑核一个worker。调度器将tasks和actors分配给workers
* Raylet：中间件，负责节点通信、分布式对象存储
* GCS：运行在某个节点上。k-v存储，包括数据对象的位置，actors的位置。

GCS存放着系统关键的元数据，在容错和性能方面很可能是系统的瓶颈，希望能够减小对它的通信频率。全局调度器同理。0.5版本移除全局调度器，0.8版本元数据去中心化（使用类似Rust的所有权机制）每个worker进程管理自己的计算任务和数据对象。

所有权机制：

![Ownership]({{ site.url }}{{ site.baseurl }}/assets/images/ownership.png){: .align-center}

* 所有权目前不能转移。
* fate-share，数据对象和它的拥有者形成绑定。

## 参考资料

Moritz, Philipp, et al. "Ray: A distributed framework for emerging {AI} applications." 13th {USENIX} Symposium on Operating Systems Design and Implementation ({OSDI} 18). 2018.

1.0架构文档 [https://docs.google.com/document/d/1lAy0Owi-vPz2jEqBSaHNQcy2IBSDEHyXNOQZlGuj93c/preview#](https://docs.google.com/document/d/1lAy0Owi-vPz2jEqBSaHNQcy2IBSDEHyXNOQZlGuj93c/preview#)

0.8架构调度优化介绍 [https://medium.com/distributed-computing-with-ray/how-ray-uses-grpc-and-arrow-to-outperform-grpc-43ec368cb385](https://medium.com/distributed-computing-with-ray/how-ray-uses-grpc-and-arrow-to-outperform-grpc-43ec368cb385)

## 任务调度

研究重点：中间件Raylet, [https://github.com/ray-project/ray/tree/master/src/ray/raylet](https://github.com/ray-project/ray/tree/master/src/ray/raylet):

* Local Scheduler：任务优先在本地执行，其次和global scheduler通信，全局调度到远端执行。
* Node Manager：负责节点之间的通信，IO模型 asio: [http://spiritsaway.info/asio-implementation.html](http://spiritsaway.info/asio-implementation.html)
* Object Manager：plasma -> Apache Arrow，**数据序列化，基于内存的对象存储**；单机内共享内存(zero-copy)模型，远程拉取数据（函数参数）到本地执行，拉取大块数据使用多线程gPRC。
* Lineage Cache: 参考Spark RDD，无状态、不可变；重新执行以恢复。

### 论文-0.5版本之前：

Bottom-up, distributed scheduler: 每个节点有一个本地调度器，某些节点上有全局调度器。

![Deprecated Schduler System]({{ site.url }}{{ site.baseurl }}/assets/images/old_scheduler.png){: .align-center}

Driver程序提交任务之后，总是先尝试在本地执行。如果本地不能满足调度所需的资源，就转发到全局的调度器。全局调度器可以通过和GCS的心跳信息来获得集群的整体状况，然后选择一个比较适合的节点进行调度。

### 0.5版本之后：

改为点对点直接进行调度 **（怎么实现？）**，看来全局调度器可能是比较严重的瓶颈？

* Merging the “local scheduler” and the “plasma manager” into a single “raylet” process (consisting of a “node manager” and an “object manager”).
* The **removal of the global scheduler** (which is replaced by direct node-manager to node-manager communication)

Lease表示以一个任务已经属于这个worker了，可以作为重新执行的依据。Lease也是调度优化的一个依据，可以在一个时间段里面将多个相似任务调度给一个拥有lease的worker，以降低调度开销。

分布式调度实现：所有节点通过心跳信息的方式通知GCS当前的资源情况（默认是100ms以减小GCS的负担），GCS再执行周期性广播。使得所有节点能够看到全局的资源情况，由于时延非常长，资源情况往往是不太准确的。于是Ray采用**spillback策略**，如果一个节点得到了调度，却刚好没有了足够的资源，它会把这个调度甩给另一个它认为可行的节点。如果当时它认为没有任何节点可以执行，将任务放在本地的队列中等待。

![New Scheduler System]({{ site.url }}{{ site.baseurl }}/assets/images/new_shceduler.png){: .align-center}

### 0.8 版本之后：

Arrow加入并行gRPC来加强节点之间的数据传输。

![Object Transfer]({{ site.url }}{{ site.baseurl }}/assets/images/object%20shceduling.png){: .align-center}

分布式对象存储，节点通过引用来控制数据，而不需要主动存放在本地。task需要数据的时候才拉取到本地。

* 小型对象（<100k），在worker进程内存储。直接附在任务描述中一起传输
* 大型对象（>100k），在Raylet的Arrow存储中，和任务调度分离

## 并行训练、仿真

```python
futures = [double.remote(i) for i in range(1000)]
ray.get(futures) # [0, 2, 4, 6, …]
```

通过caching的方式来平衡并行计算的开销：Raylet缓存函数的调度决定，在初次调度之后的一小段时间内，直接将相同函数的其他调用扔给之前就在执行的worker。这个时间段不会很长，防止driver把所有调度塞给一个worker。

![Concurrent Scheduling]({{ site.url }}{{ site.baseurl }}/assets/images/concurrent%20execution.png){: .align-center}
