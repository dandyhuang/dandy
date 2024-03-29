# kafka基本概念

最近接到snapshot落kafka需求，kafka配置中提到了topic、group、brokers、partion等参数。一开始看很闷，只是知道kafka作为消息队列使用，但是使用它起来，很多概念都不清晰。并且封装好的库一般只需要你填写key和value即可。

### 为何使用消息队列

- **解耦**
  　　在项目启动之初来预测将来项目会碰到什么需求，是极其困难的。消息系统在处理过程中间插入了一个隐含的、基于数据的接口层，两边的处理过程都要实现这一接口。这允许你独立的扩展或修改两边的处理过程，只要确保它们遵守同样的接口约束。
- **冗余**
  　　有些情况下，处理数据的过程会失败。除非数据被持久化，否则将造成丢失。消息队列把数据进行持久化直到它们已经被完全处理，通过这一方式规避了数据丢失风险。许多消息队列所采用的”插入-获取-删除”范式中，在把一个消息从队列中删除之前，需要你的处理系统明确的指出该消息已经被处理完毕，从而确保你的数据被安全的保存直到你使用完毕。
- **扩展性**
  　　因为消息队列解耦了你的处理过程，所以增大消息入队和处理的频率是很容易的，只要另外增加处理过程即可。不需要改变代码、不需要调节参数。扩展就像调大电力按钮一样简单。
- **灵活性 & 峰值处理能力**
  　　在访问量剧增的情况下，应用仍然需要继续发挥作用，但是这样的突发流量并不常见；如果为以能处理这类峰值访问为标准来投入资源随时待命无疑是巨大的浪费。使用消息队列能够使关键组件顶住突发的访问压力，而不会因为突发的超负荷的请求而完全崩溃。
- **可恢复性**
  　　系统的一部分组件失效时，不会影响到整个系统。消息队列降低了进程间的耦合度，所以即使一个处理消息的进程挂掉，加入队列中的消息仍然可以在系统恢复后被处理。
- **顺序保证**
  　　在大多使用场景下，数据处理的顺序都很重要。大部分消息队列本来就是排序的，并且能保证数据会按照特定的顺序来处理。Kafka保证一个Partition内的消息的有序性。
- **缓冲**
  　　在任何重要的系统中，都会有需要不同的处理时间的元素。例如，加载一张图片比应用过滤器花费更少的时间。消息队列通过一个缓冲层来帮助任务最高效率的执行———写入队列的处理会尽可能的快速。该缓冲有助于控制和优化数据流经过系统的速度。
- **异步通信**
  　　很多时候，用户不想也不需要立即处理消息。消息队列提供了异步处理机制，允许用户把一个消息放入队列，但并不立即处理它。想向队列中放入多少消息就放多少，然后在需要的时候再去处理它们。

### kafka基本架构

![image-20211016161735893](/Users/11126518/knowledge/dandy_blog/image/kafka架构图.png)

kafka由Producer、Broker、Consumer三大模块组成，我们知道kafka是作为消息队列使用，Producer就是对应生产者，而Consumer对应的就是消费者。这两者都需要使用方根据业务编写对应的客户端代码。Broker对应服务端，接收来找Producer和Consumer的请求，并保持或发送相应数据。每个kafka集群对应一个或多个Broker。

- Producer使用push模式将消息发布到broker的客户端
- Consumer使用pull模式从broker订阅并消费消息的客户端
- Broker消息中间件处理节点；

### 每个broker可以理解为一个redis cluster

- redis 集群包含 16384 个哈希槽，每个槽对应分配好的节点。而每个broker里头，partition可以看成是哈希槽分配后的节点。
- redis扩容增加节点，需要修改槽信息映射关系，并且还需要数据迁移。而broker扩容只需要增加partition分区。
- redis cluser节点也有主从模式，并且也是主对外提供交互。而partition中也有Leader和Replication，Producer和Consumer也只与这个Leader交互。
- 主从数据一致性方面也是一样提供了三只模式，只管发送、主节点接收到、或者Leader和Replication全部都接收到。
- redis cluser选主是基于raft协议，而partion里头的选主是有zookeepr来管理的，zookeeper基于Zab协议

对于topic的理解，可以任务是Kafka集群下有多个redis cluser吧。每个topic可以被多个消费组消费。