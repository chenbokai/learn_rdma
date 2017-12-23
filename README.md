# learn_rdma
search rdma



Yahoo! 近日在开源 TensorFlowOnSpark 的同时，开放了其 TensorFlow 增强版本，其中一个重要特性是在对 RDMA 通信协议的支持。

 

TensorFlowOnSpark 及 RDMA 相关支持的介绍文章：

  * http://yahoohadoop.tumblr.com/post/157196317141/open-sourcing-tensorflowonspark-distributed-deep （官方）

  * http://www.infoq.com/cn/news/2017/02/Spark-Yahoo-TensorFlowOnSpark （中文翻译）

 

文章指出，目前的 RDMA 支持是 alpha 版本，近几周内会进一步完善，并将发布 benchmark 结果。Yahoo! 期望将其推入 TensorFlow 社区。

 

* 作者在社区的说法

 

从作者 Jun Shi 在 Github （https://github.com/tensorflow/tensorflow/issues/2916）上的说明看，他们做到的加速效果：大规模 VGG_19 速度为 2x（相当于加速 100%）；2 节点 inception_v3 加速 10%。他们对比的对象是“ethernet”，不知道是指 IPoIB 还是 10Gbps 以太网（40Gbps 的 IB 卡跑 IPoIB 时，性能一般和 10Gpbs 的以太网卡相当）。

 

作者 Andy Feng 在 Github 上提到，他们不用 MPI 而直接用 RDMA 的原因是“It is designed to work in a cloud setting, where a set of nodes are dynamically allocated. That's why we avoid MPI option.”这个说法是有道理的。因为 TensorFlow 事实上是允许同一作业的 worker 在运行时动态增减的，用 MPI 的确限制了 TensorFlow 单一作业的动态伸缩性和容错性。

 

作者 Junshi 提到，当前的 Yahoo! RDMA 工作是基于 TensorFlow 0.12.1 做的。

 

* 我对 Yahoo! RDMA 源代码的初步分析

 

源代码在这里： https://github.com/yahoo/tensorflow/tree/yahoo/tensorflow/core/distributed_runtime/rdma

官方说明文档： https://github.com/yahoo/tensorflow/blob/yahoo/tensorflow/core/distributed_runtime/rdma/rdma_readme.md

 

他们的整体思路是：保留 gRPC session 机制，新增一种与 gRPC 兼容的 RdmaRendezvousMgr 和 RDMA 通信管理类，使用原有的 gRPC 调度机制调度 RDMA 通信。理论上说，gRPC 通信和 RDMA 通信应该是并列的机制，RDMA 不需要依赖于 gRPC。但 TensorFlow 的通信与调度是高度耦合的，分布式调度机制依赖于 gRPC 的队列机制。所以，RDMA 通信的设计脱离不了 gRPC 原有的调度机制。

 

他们只使用 RDMA 接管数据面通信（RecvTensor），不替代控制面通信。他们新增了一种 GetRemoteAddress RPC 接口，用来在 RDMA 通信开始之前传递 RDMA 通信所需的元信息。所以，他们是把一次 RecvTensor RPC 分解为一次 GetRemoteAddress RPC + 一次 RDMA。

 

他们设计了一套完善的内部缓冲区管理 + RDMA 通知/反馈消息机制。这样做的好处是：避免了 RDMA 消息对 RPC 消息的等待，提高了通信和计算的并发度。缺点在于：（1）增加了本地内存复制开销，部分地抵销了 RDMA“内存零复制、CPU 零参与”的优势；（2）增加了 RDMA 通知/反馈消息的通信和等待开销。
