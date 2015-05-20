# kafka学习笔记 - 使用与配置

## 一. 使用

1. 下载代码

	[https://www.apache.org/dyn/closer.cgi?path=/kafka/0.8.2.0/kafka_2.10-0.8.2.0.tgz](https://www.apache.org/dyn/closer.cgi?path=/kafka/0.8.2.0/kafka_2.10-0.8.2.0.tgz)

		tar -xzf kafka_2.10-0.8.2.0.tgz
		cd kafka_2.10-0.8.2.0
		
2. 启动服务器

	kafka依赖zookeeper，所以需要首先安装并启动zookeeper。可以使用kafka自带的zookeeper。
	
		bin/zookeeper-server-start.sh config/zookeeper.properties
		
	然后即可启动kafka
	
		bin/kafka-server-start.sh config/server.properties
		
3. 创建topic

	消息传输需要指定topic。所以首先要创建一个topic。
	
		bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
	
	之后，可以看到已经创建的topic.其中的replication-factor指的是复制因子，即log冗余的份数，这里的数字不能大于broker的数量。
	
		bin/kafka-topics.sh --list --zookeeper localhost:2181
		
	也可以不用手动创建topic，只需要配置broker的时候设置为auto-create topic when a non-existent topic is published to.
	
4. 发送消息

	kafka提供了一个命令行客户端，可以从一个文件或者标准输入里读取并发送到kafka集群。默认的，每一行都作为一个单独的消息。
	
		bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test 
		
	在命令行输入消息并回车即可发送消息。
	
5. 启动一个消费者

	kafka也提供了一个命令行消费者，接受消息并打印到标准输出。
	
		bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic test --from-beginning
	
6. 设置多broker集群
	
	首先需要为每一个broker创建一个配置文件。
	
		cp config/server.properties config/server-1.properties 
 		cp config/server.properties config/server-2.properties
 		
 		config/server-1.properties:
    		broker.id=1
    		port=9093
    		log.dir=/tmp/kafka-logs-1
 
		config/server-2.properties:
    		broker.id=2
    		port=9094
    	log.dir=/tmp/kafka-logs-2	
    	
    然后启动这两个结点：
    
    	bin/kafka-server-start.sh config/server-1.properties &
    	bin/kafka-server-start.sh config/server-2.properties &

    现在一共有了三个结点，三个broker，那么这样就可以形成一个集群。
    
    创建一个复制引子为3的topic
    
    	bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic my-replicated-topic
    	
    如果想查看目前这个topic的partion在broker上的分布情况
    
    	bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic my-replicated-topic
    	
## 二. 关键配置

### 2.1 broker

- broker.id
- log.dirs
- zookeeper.connect

### 2.2 Producter

- metadata.broker.list
- request.required.acks
- producer.type
- serializer.class