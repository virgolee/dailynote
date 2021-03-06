# RabbitMQ总结

## 1. 使用RabbitMQ有什么好处？

- 应用解耦（系统拆分）
- 异步处理（预约挂号业务处理成功后，异步发送短信、推送消息、日志记录等）
- 消息分发
- 流量削峰
- 消息缓冲

## 2. 消息基于什么传输？

RabbitMQ消息基于信道传输。

由于TCP连接的创建和销毁开销较大(三次握手，四次挥手)，且并发数受系统资源限制，会造成性能瓶颈。RabbitMQ使用信道的方式来传输数据。信道是建立在真实的TCP连接内的虚拟连接，且每条TCP连接上的信道数量没有限制。

## 3. 消息怎么路由？

从概念上来说，消息路由必须有三部分：**交换器、路由、绑定**。生产者把消息发布到交换器上；绑定决定了消息如何从路由器路由到特定的队列；消息最终到达队列，并被消费者接收。常用的交换器主要分为一下三种：

- direct：如果路由键完全匹配，消息就被投递到相应的队列
- fanout：如果交换器收到消息，将会广播到所有绑定的队列上
- topic：可以使来自不同源头的消息能够到达同一个队列。 使用topic交换器时，可以使用通配符，比如：“*” 匹配特定位置的任意文本， “.” 把路由键分为了几部分，“#” 匹配所有规则等。特别注意：发往topic交换器的消息不能随意的设置选择键（routing_key），必须是由"."隔开的一系列的标识符组成。

## 4. 如何做到信息的可靠性？确保消息正确地发送至RabbitMQ？确保消息接受方消费了消息？休息不丢失不重复？

采用事务或发送方确认模式保证消息发送到rabbitmq。 

生产者发送消息到MQ，MQ回复ACK确认收到消息。

MQ将消息发送给消费者，消费者收到后回复ACK确认收到消息。

### 发送方确认模式

将信道设置成confirm模式（发送方确认模式），则所有在信道上发布的消息都会被指派一个唯一的ID。一旦消息被投递到目的队列后，或者消息被写入磁盘后（可持久化的消息），信道会发送一个确认给生产者（包含消息唯一ID）。如果RabbitMQ发生内部错误从而导致消息丢失，会发送一条nack（not acknowledged，未确认）消息。发送方确认模式是异步的，生产者应用程序在等待确认的同时，可以继续发送消息。当确认消息到达生产者应用程序，生产者应用程序的回调方法就会被触发来处理确认消息。

### 接收方消息确认机制

消费者接收每一条消息后都必须进行确认（消息接收和消息确认是两个不同操作）。只有消费者确认了消息，RabbitMQ才能安全地把消息从队列中删除。RabbitMQ仅通过Consumer的连接中断来确认是否需要重新发送消息。也就是说，只要连接不中断，RabbitMQ给了Consumer足够长的时间来处理消息。消费者接收到消息，在确认之前断开了连接或取消订阅，RabbitMQ会认为消息没有被分发，然后重新分发给下一个订阅的消费者。（可能存在消息重复消费的隐患，需要根据bizId去重）

### 消息去重

 在消息生产时，MQ内部针对每条生产者发送的消息生成一个inner-msg-id，作为去重和幂等的依据（消息投递失败并重传），避免重复的消息进入队列；在消息消费时，要求消息体中必须要有一个bizId（对于同一业务全局唯一，如支付ID、订单ID、帖子ID等）作为去重和幂等的依据，避免同一条消息被重复消费。

## 5. vhost 是什么？起什么作用？

vhost 可以理解为虚拟 broker ，即 mini-RabbitMQ  server。其内部均含有独立的 queue、exchange 和 binding 等，但最最重要的是，其拥有独立的权限系统，可以做到 vhost 范围的用户控制。当然，从 RabbitMQ 的全局角度，vhost 可以作为不同权限隔离的手段（一个典型的例子就是不同的应用可以跑在不同的 vhost 中）。 

## 6. 为什么使用集群？

内建集群作为RabbitMQ最优秀的功能之一，它的作用有两个：

1. 允许消费者和生产者在Rabbit节点崩溃的情况下继续运行；
2. 通过增加节点来扩展Rabbit处理更多的消息，承载更多的业务量；

RabbitMQ的集群是由多个节点组成的，但我们发现不是每个节点都有所有队列的完全拷贝。

### RabbitMQ节点不完全拷贝特性

为什么默认情况下RabbitMQ不将所有队列内容和状态复制到所有节点？

有两个原因：

1. 存储空间——如果每个节点都拥有所有队列的完全拷贝，这样新增节点不但没有新增存储空间，反而增加了更多的冗余数据。
2. 性能——如果消息的发布需安全拷贝到每一个集群节点，那么新增节点对网络和磁盘负载都会有增加，这样违背了建立集群的初衷，新增节点并没有提升处理消息的能力，最多是保持和单节点相同的性能甚至是更糟。

所以其他非所有者节点只知道队列的元数据，和指向该队列节点的指针。根据节点不无安全拷贝的特性，当集群节点崩溃时，该节点队列和关联的绑定就都丢失了，附加在该队列的消费者丢失了其订阅的信息，那么怎么处理这个问题呢？

这个问题要分为两种情况：

1. 消息已经进行了持久化，那么当节点恢复，消息也恢复了；
2. 消息未持久化，可以使用下文要介绍的双活冗余队列，镜像队列保证消息的可靠性；

### 镜像队列

镜像队列是Rabbit2.6.0版本带来的一个新功能，允许内建双活冗余选项，与普通队列不同，镜像节点在集群中的其他节点拥有从队列拷贝，一旦主节点不可用，最老的从队列将被选举为新的主队列。

**镜像队列的工作原理：**在某种程度上你可以将镜像队列视为，拥有一个隐藏的fanout交换器，它指示者信道将消息分发到从队列上。

#### 设置镜像队列

设置镜像队列命令：“rabbitmqctl set_policy 名称 匹配模式（正则） 镜像定义”， 例如，设置名称为mypolicy的镜像队列，匹配所有名称是amp开头的队列都存储在2个节点上的命令如下：

> rabbitmqctl set_policy mypolicy "^amp*" '{"ha-mode":"exactly","ha-params":2}'

可以看出**设置镜像队列，一共有三个参数，每个参数用空格分割**。

1. 参数一：名称，可以随便填；
2. 参数二：队列名称的匹配规则，使用正则表达式表示；
3. 参数三：为镜像队列的主体规则，是json字符串，分为三个属性：ha-mode | ha-params | ha-sync-mode，分别的解释如下：

- ha-mode：镜像模式，分类：all/exactly/nodes，all存储在所有节点；exactly存储x个节点，节点的个数由ha-params指定；nodes指定存储的节点上名称，通过ha-params指定；
- ha-params：作为参数，为ha-mode的补充；
- ha-sync-mode：镜像消息同步方式：automatic（自动），manually（手动）；

## 集群节点类型

节点的存储类型分为两种：

- 磁盘节点
- 内存节点

磁盘节点就是配置信息和元信息存储在磁盘上，内次节点把这些信息存储在内存中，当然内次节点的性能是大大超越磁盘节点的。

**单节点系统必须是磁盘节点**，否则每次你重启RabbitMQ之后所有的系统配置信息都会丢失。

**RabbitMQ要求集群中至少有一个磁盘节点**，当节点加入和离开集群时，必须通知磁盘节点。

### 特殊异常：集群中唯一的磁盘节点崩溃了

如果集群中的唯一一个磁盘节点崩溃了，那会发生什么情况？

如果唯一磁盘的磁盘节点崩溃了，不能进行如下操作：

- 不能创建队列
- 不能创建交换器
- 不能创建绑定
- 不能添加用户
- 不能更改权限
- 不能添加和删除集群几点

**总结：如果唯一磁盘的磁盘节点崩溃，集群是可以保持运行的，但你不能更改任何东西。**

**解决方案：**在集群中设置两个磁盘节点，只要一个可以，你就能正常操作。

