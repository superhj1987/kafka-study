# Kafka Consumer开发的一些关键点

Kafka的consumer是以pull的形式获取消息数据的。不同于队列和发布-订阅模式，kafka采用了consumer group的模式。通常的，一般采用一个consumer中的一个group对应一个业务，配合多个producer提供数据。

![pic](http://static.oschina.net/uploads/space/2013/0225/222954_DNR2_589742.jpg)

## 一. 消费过的数据无法再次消费

在user level上，一旦消费过topic里的数据，那么就无法再次用同一个groupid消费同一组数据。如果想要再次消费数据，要么换另一个groupid，要么使用镜像：
	
![pic](http://static.oschina.net/uploads/space/2013/0225/223246_m87S_589742.jpg)

此外，low level的api提供了一些机制去设置partion和offset。

## 二. offset管理

kafka会记录offset到zk中。但是，zk client api对zk的频繁写入是一个低效的操作。0.8.2 kafka引入了native offset storage，将offset管理从zk移出，并且可以做到水平扩展。其原理就是利用了kafka的compacted topic，offset以consumer group,topic与partion的组合作为key直接提交到compacted topic中。同时Kafka又在内存中维护了<consumer group,topic,partition>的三元组来维护最新的offset信息，consumer来取最新offset信息的时候直接内存里拿即可。当然，kafka允许你快速的checkpoint最新的offset信息到磁盘上。

## 三. stream

This API is centered around iterators, implemented by the KafkaStream class. Each KafkaStream represents the stream of messages from one or more partitions on one or more servers. Each stream is used for single threaded processing, so the client can provide the number of desired streams in the create call. Thus a stream may represent the merging of multiple server partitions (to correspond to the number of processing threads), but each partition only goes to one stream.

根据官方文档所说，stream即指的是来自一个或多个服务器上的一个或者多个partition的消息。每一个stream都对应一个单线程处理。因此，client能够设置满足自己需求的stream数目。总之，一个stream也许代表了多个服务器partion的消息的聚合，但是每一个partition都只能到一个stream。

## 四. consumer和partition

1. 如果consumer比partition多，是浪费，因为kafka的设计是在一个partition上是不允许并发的，所以consumer数不要大于partition数 
2. 如果consumer比partition少，一个consumer会对应于多个partitions，这里主要合理分配consumer数和partition数，否则会导致partition里面的数据被取的不均匀 
3. 如果consumer从多个partition读到数据，不保证数据间的顺序性，kafka只保证在一个partition上数据是有序的，但多个partition，根据你读的顺序会有不同 
4. 增减consumer，broker，partition会导致rebalance，所以rebalance后consumer对应的partition会发生变化 
5. High-level接口中获取不到数据的时候是会block的

负载低的情况下可以每个线程消费多个partition。但负载高的情况下，Consumer 线程数最好和Partition数量保持一致。如果还是消费不过来，应该再开 Consumer 进程，进程内线程数同样和分区数一致。（多谢 @shadyxu 指出）

## 五. high-level的consumer工具

1. bin/kafka-run-class.sh kafka.tools.ConsumerOffsetChecker --group pv

	可以看到当前group offset的状况。

2. bin/kafka-run-class.sh kafka.tools.UpdateOffsetsInZK earliest config/consumer.properties  page_visits

		3个参数， 
		[earliest | latest]，表示将offset置到哪里 
		consumer.properties ，这里是配置文件的路径 
		topic，topic名，这里是page_visits
		
## 六. SimpleConsumer

kafka的low-level接口，使用场景：

1. Read a message multiple times
2. Consume only a subset of the partitions in a topic in a process
3. Manage transactions to make sure a message is processed once and only once

用这个接口需要注意：

1. You must keep track of the offsets in your application to know where you left off consuming.
2. You must figure out which Broker is the lead Broker for a topic and partition
3. You must handle Broker leader changes

使用步骤：

1. Find an active Broker and find out which Broker is the leader for your topic and partition：你必须知道读哪个topic的哪个partition 
2. Determine who the replica Brokers are for your topic and partition： 找到负责该partition的broker leader，从而找到存有该partition副本的那个broker
3. Build the request defining what data you are interested in：自己去写request并fetch数据 
4. Fetch the data
5. Identify and recover from leader changes：还要注意需要识别和处理broker leader的改变

