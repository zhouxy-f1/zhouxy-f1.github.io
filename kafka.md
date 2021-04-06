## 一、安装部署

##### 架构原理

```bash
#消息系统
一个消息系统负责将数据从一个应用传递到另外一个应用，应用只需关注于数据，无需关注数据在两个或多个应用间是如何传递的。分布式消息传递基于可靠的消息队列，在客户端应用和消息系统之间异步传递消息

#消息传递模式
点对点模式：发布者发送信息到队列里，消费者主动拉取消息，消费后从队列中删除，一条消息只能被消费一次
发布/订阅：发布者发送信息到topic，多个消费者订阅topic，消费后不会删除，信息在队列里的保存有时限，有两种模式 
1.消费者主动拉取：如kafka，各消费者需要维护一个长轮询来查看队列里有没有新消息
2.由队列推送信息：推送消息的速度是由队列决定的，但消费者接收处理的速度不同，性能不好的消费者机器可能扛不住

#kafka架构
kafka是分布式的基于发布/订阅模式的消息队列

1.Producer: 消息生产者，向broker发消息
2.Consumer: 消息消费者，从broker读消息
3.Consumer Group: 消费者组，由多个consumer组成，一个组内每个消费者消费不同分区的数据，一个分区只能由一个组内消费者消费，消费者之间互不影响

#主体
Broker: kafka集群包含多台服务器，用broker0、broker1来标识各节点
Producer: 生产者即数据的发布者
Consumer: 消费者即数据的订阅者
Consumer Group: 消费者组，每个消费者都要有特定的组，未指定的则属于默认组，组内

#topic、partition
消息系统需要处理多种信息，把信息分类处理，这个类别就是topic，把不同机器上的存储切分为partition0~n，partition0专用于存topicA的消息，不同机器上的partition是主从关系，主是leader，从是follower

#Leader与follower
partition有多个副本时，leader是当前负责数据读写的partition，follower与leader同步数据，当leader挂了，从follower中选举新的leader

Zookeeper：用于组件kafka集群，0.9版本之前offset存储在zookeeper,新版本存到了系统topic里，存7天  

```

##### 安装部署

```bash
#下载
wget https://mirrors.tuna.tsinghua.edu.cn/apache/kafka/2.6.0/kafka_2.12-2.6.0.tgz
tar xf kafka_2.12-2.6.0.tgz -C /opt
mv /opt/kafka_2.12-2.6.0 /opt/kafka

#配置
vim /opt/kafka/config/server.properties
broker.id=0														#集群中本节点的唯一标识
log.dirs=/tmp/kafka-logs							#消息数据存放目录
log.retention.hours=168								#消息数据保留多久
zookeeper.connect=localhost:2181			#zk连接信息
vim /opt/kafka/config/zookeeper.properties

zookeeper-server-start.sh -daemon /opt/kafka/config/zookeeper.properties
kafka-server-start.sh -daemon /opt/kafka/config/server.properties

#环境变量
cat >>/etc/profile<<'EOF'
export KAFKA_HOME=/opt/kafka
export PATH=$PATH:$KAFKA_HOME/bin
EOF
source /etc/profile
```

##### 基础命令

```bash
#topic
kafka-topic.sh --zookeeper server1:2181 --list
kafka-topic.sh --zookeeper server1:2181 --create --topic test1 --replication-factor 3 --partition 1
kafka-topic.sh --zookeeper server1:2181 --delete --topic test1
```

