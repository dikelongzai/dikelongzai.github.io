---
title: Kafka设计解析-Kafka High Availability
categories:
  - 大数据
  - 消息队列
  
tags:
  - kafka
abbrlink: 33760
date: 2018-04-03 01:29:56
---
Kafka在0.8以前的版本中，并不提供High Availablity机制，一旦一个或多个Broker宕机，则宕机期间其上所有Partition都无法继续提供服务。若该Broker永远不能再恢 复，亦或磁盘故障，则其上数据将丢失。而Kafka的设计目标之一即是提供数据持久化，同时对于分布式系统来说，尤其当集群规模上升到一定程度后，一台或 者多台机器宕机的可能性大大提高，对Failover要求非常高。因此，Kafka从0.8开始提供High Availability机制。本文从Data Replication和Leader Election两方面介绍了Kafka的HA机制。
# Kafka为何需要High Available
## 为何需要Replication
在Kafka在0.8以前的版本中，是没有Replication的，一旦某一个Broker宕机，则其上所有的Partition数据都不可被消 费，这与Kafka数据持久性及Delivery Guarantee的设计目标相悖。同时Producer都不能再将数据存于这些Partition中。
如果Producer使用同步模式则Producer会在尝试重新发送message.send.max.retries（默认值为3）次后抛出Exception，用户可以选择停止发送后续数据也可选择继续选择发送。而前者会造成数据的阻塞，后者会造成本应发往该Broker的数据的丢失。
如果Producer使用异步模式，则Producer会尝试重新发送message.send.max.retries（默认值为3）次后记录该异常并继续发送后续数据，这会造成数据丢失并且用户只能通过日志发现该问题。同时，Kafka的Producer并未对异步模式提供callback接口。
由此可见，在没有Replication的情况下，一旦某机器宕机或者某个Broker停止工作则会造成整个系统的可用性降低。随着集群规模的增加，整个集群中出现该类异常的几率大大增加，因此对于生产系统而言Replication机制的引入非常重要。
## 为何需要Leader Election
注意：本文所述Leader Election主要指Replica之间的Leader Election。
引入Replication之后，同一个Partition可能会有多个Replica，而这时需要在这些Replication之间选出一个 Leader，Producer和Consumer只与这个Leader交互，其它Replica作为Follower从Leader中复制数据。
因为需要保证同一个Partition的多个Replica之间的数据一致性（其中一个宕机后其它Replica必须要能继续服务并且即不能造成数 据重复也不能造成数据丢失）。如果没有一个Leader，所有Replica都可同时读/写数据，那就需要保证多个Replica之间互相（N×N条通 路）同步数据，数据的一致性和有序性非常难保证，大大增加了Replication实现的复杂性，同时也增加了出现异常的几率。而引入Leader后，只 有Leader负责数据读写，Follower只向Leader顺序Fetch数据（N条通路），系统更加简单且高效。
# Kafka HA设计解析
## 如何将所有Replica均匀分布到整个集群
为了更好的做负载均衡，Kafka尽量将所有的Partition均匀分配到整个集群上。一个典型的部署方式是一个Topic的Partition 数量大于Broker的数量。同时为了提高Kafka的容错能力，也需要将同一个Partition的Replica尽量分散到不同的机器。实际上，如果 所有的Replica都在同一个Broker上，那一旦该Broker宕机，该Partition的所有Replica都无法工作，也就达不到HA的效 果。同时，如果某个Broker宕机了，需要保证它上面的负载可以被均匀的分配到其它幸存的所有Broker上。
Kafka分配Replica的算法如下：
将所有Broker（假设共n个Broker）和待分配的Partition排序
将第i个Partition分配到第（i mod n）个Broker上
将第i个Partition的第j个Replica分配到第（(i + j) mode n）个Broker上
## Data Replication
Kafka的Data Replication需要解决如下问题：
## 怎样Propagate消息
在向Producer发送ACK前需要保证有多少个Replica已经收到该消息
怎样处理某个Replica不工作的情况
怎样处理Failed Replica恢复回来的情况
Propagate消息
Producer在发布消息到某个Partition时，先通过ZooKeeper找到该Partition的Leader，然后无论该Topic 的Replication Factor为多少（也即该Partition有多少个Replica），Producer只将该消息发送到该Partition的Leader。 Leader会将该消息写入其本地Log。每个Follower都从Leader pull数据。这种方式上，Follower存储的数据顺序与Leader保持一致。Follower在收到该消息并写入其Log后，向Leader发送 ACK。一旦Leader收到了ISR中的所有Replica的ACK，该消息就被认为已经commit了，Leader将增加HW并且向 Producer发送ACK。
为了提高性能，每个Follower在接收到数据后就立马向Leader发送ACK，而非等到数据写入Log中。因此，对于已经commit的消 息，Kafka只能保证它被存于多个Replica的内存中，而不能保证它们被持久化到磁盘中，也就不能完全保证异常发生后该条消息一定能被 Consumer消费。但考虑到这种场景非常少见，可以认为这种方式在性能和数据持久化上做了一个比较好的平衡。在将来的版本中，Kafka会考虑提供更 高的持久性。
Consumer读消息也是从Leader读取，只有被commit过的消息（offset低于HW的消息）才会暴露给Consumer。
Kafka Replication的数据流如下图所示：
![kafka1：spark血缘](/public/image/kafka-1.png)
## ACK前需要保证有多少个备份
和大部分分布式系统一样，Kafka处理失败需要明确定义一个Broker是否“活着”。对于Kafka而言，Kafka存活包含两个条件，一是它 必须维护与ZooKeeper的session（这个通过ZooKeeper的Heartbeat机制来实现）。二是Follower必须能够及时将 Leader的消息复制过来，不能“落后太多”。
Leader会跟踪与其保持同步的Replica列表，该列表称为ISR（即in-sync Replica）。如果一个Follower宕机，或者落后太多，Leader将把它从ISR中移除。这里所描述的“落后太多”指Follower复制的 消息落后于Leader后的条数超过预定值（该值可在$KAFKA_HOME/config/server.properties中通过replica.lag.max.messages配置，其默认值是4000）或者Follower超过一定时间（该值可在$KAFKA_HOME/config/server.properties中通过replica.lag.time.max.ms来配置，其默认值是10000）未向Leader发送fetch请求。
Kafka的复制机制既不是完全的同步复制，也不是单纯的异步复制。事实上，完全同步复制要求所有能工作的Follower都复制完，这条消息才会 被认为commit，这种复制方式极大的影响了吞吐率（高吞吐率是Kafka非常重要的一个特性）。而异步复制方式下，Follower异步的从 Leader复制数据，数据只要被Leader写入log就被认为已经commit，这种情况下如果Follower都复制完都落后于Leader，而如 果Leader突然宕机，则会丢失数据。而Kafka的这种使用ISR的方式则很好的均衡了确保数据不丢失以及吞吐率。Follower可以批量的从 Leader复制数据，这样极大的提高复制性能（批量写磁盘），极大减少了Follower与Leader的差距。
需要说明的是，Kafka只解决fail/recover，不处理“Byzantine”（“拜占庭”）问题。一条消息只有被ISR里的所有 Follower都从Leader复制过去才会被认为已提交。这样就避免了部分数据被写进了Leader，还没来得及被任何Follower复制就宕机 了，而造成数据丢失（Consumer无法消费这些数据）。而对于Producer而言，它可以选择是否等待消息commit，这可以通过request.required.acks来设置。这种机制确保了只要ISR有一个或以上的Follower，一条被commit的消息就不会丢失。
## Leader Election算法
上文说明了Kafka是如何做Replication的，另外一个很重要的问题是当Leader宕机了，怎样在Follower中选举出新的 Leader。因为Follower可能落后许多或者crash了，所以必须确保选择“最新”的Follower作为新的Leader。一个基本的原则就 是，如果Leader不在了，新的Leader必须拥有原来的Leader commit过的所有消息。这就需要作一个折衷，如果Leader在标明一条消息被commit前等待更多的Follower确认，那在它宕机之后就有更 多的Follower可以作为新的Leader，但这也会造成吞吐率的下降。
一种非常常用的选举leader的方式是“Majority Vote”（“少数服从多数”），但Kafka并未采用这种方式。这种模式下，如果我们有2f+1个Replica（包含Leader和 Follower），那在commit之前必须保证有f+1个Replica复制完消息，为了保证正确选出新的Leader，fail的Replica不 能超过f个。因为在剩下的任意f+1个Replica里，至少有一个Replica包含有最新的所有消息。这种方式有个很大的优势，系统的latency 只取决于最快的几个Broker，而非最慢那个。Majority Vote也有一些劣势，为了保证Leader Election的正常进行，它所能容忍的fail的follower个数比较少。如果要容忍1个follower挂掉，必须要有3个以上的 Replica，如果要容忍2个Follower挂掉，必须要有5个以上的Replica。也就是说，在生产环境下为了保证较高的容错程度，必须要有大量 的Replica，而大量的Replica又会在大数据量下导致性能的急剧下降。这就是这种算法更多用在ZooKeeper这种共享集群配置的系统中而很少在需要存储大量数据的系统中使用的原因。例如HDFS的HA Feature是基于majority-vote-based journal，但是它的数据存储并没有使用这种方式。
实际上，Leader Election算法非常多，比如ZooKeeper的Zab, Raft和Viewstamped Replication。而Kafka所使用的Leader Election算法更像微软的PacificA算法。
Kafka在ZooKeeper中动态维护了一个ISR（in-sync replicas），这个ISR里的所有Replica都跟上了leader，只有ISR里的成员才有被选为Leader的可能。在这种模式下，对于 f+1个Replica，一个Partition能在保证不丢失已经commit的消息的前提下容忍f个Replica的失败。在大多数使用场景中，这种 模式是非常有利的。事实上，为了容忍f个Replica的失败，Majority Vote和ISR在commit前需要等待的Replica数量是一样的，但是ISR需要的总的Replica的个数几乎是Majority Vote的一半。
虽然Majority Vote与ISR相比有不需等待最慢的Broker这一优势，但是Kafka作者认为Kafka可以通过Producer选择是否被commit阻塞来改善这一问题，并且节省下来的Replica和磁盘使得ISR模式仍然值得。
## 如何处理所有Replica都不工作
上文提到，在ISR中至少有一个follower时，Kafka可以确保已经commit的数据不丢失，但如果某个Partition的所有Replica都宕机了，就无法保证数据不丢失了。这种情况下有两种可行的方案：
等待ISR中的任一个Replica“活”过来，并且选它作为Leader
选择第一个“活”过来的Replica（不一定是ISR中的）作为Leader
这就需要在可用性和一致性当中作出一个简单的折衷。如果一定要等待ISR中的Replica“活”过来，那不可用的时间就可能会相对较长。而且如果 ISR中的所有Replica都无法“活”过来了，或者数据都丢失了，这个Partition将永远不可用。选择第一个“活”过来的Replica作为 Leader，而这个Replica不是ISR中的Replica，那即使它并不保证已经包含了所有已commit的消息，它也会成为Leader而作为 consumer的数据源（前文有说明，所有读写都由Leader完成）。Kafka0.8.*使用了第二种方式。根据Kafka的文档，在以后的版本 中，Kafka支持用户通过配置选择这两种方式中的一种，从而根据不同的使用场景选择高可用性还是强一致性。
## 如何选举Leader
最简单最直观的方案是，所有Follower都在ZooKeeper上设置一个Watch，一旦Leader宕机，其对应的ephemeral znode会自动删除，此时所有Follower都尝试创建该节点，而创建成功者（ZooKeeper保证只有一个能创建成功）即是新的Leader，其 它Replica即为Follower。
但是该方法会有3个问题：
split-brain 这是由ZooKeeper的特性引起的，虽然ZooKeeper能保证所有Watch按顺序触发，但并不能保证同一时刻所有Replica“看”到的状态是一样的，这就可能造成不同Replica的响应不一致
herd effect 如果宕机的那个Broker上的Partition比较多，会造成多个Watch被触发，造成集群内大量的调整
ZooKeeper负载过重 每个Replica都要为此在ZooKeeper上注册一个Watch，当集群规模增加到几千个Partition时ZooKeeper负载会过重。
Kafka 0.8.*的Leader Election方案解决了上述问题，它在所有broker中选出一个controller，所有Partition的Leader选举都由 controller决定。controller会将Leader的改变直接通过RPC的方式（比ZooKeeper Queue的方式更高效）通知需为为此作为响应的Broker。同时controller也负责增删Topic以及Replica的重新分配。
## HA相关ZooKeeper结构
首先声明本节所示ZooKeeper结构中，实线框代表路径名是固定的，而虚线框代表路径名与业务相关
admin （该目录下znode只有在有相关操作时才会存在，操作结束时会将其删除）
![kafka2](/public/image/kafka-2.png)
/admin/preferred_replica_election数据结构
```
{
   "fields":[
      {
         "name":"version",
         "type":"int",
         "doc":"version id"
      },
      {
         "name":"partitions",
         "type":{
            "type":"array",
            "items":{
               "fields":[
                  {
                     "name":"topic",
                     "type":"string",
                     "doc":"topic of the partition for which preferred replica election should be triggered"
                  },
                  {
                     "name":"partition",
                     "type":"int",
                     "doc":"the partition for which preferred replica election should be triggered"
                  }
               ],
            }
            "doc":"an array of partitions for which preferred replica election should be triggered"
         }
      }
   ]
}
```
Example:
```
{
  "version": 1,
  "partitions":
     [
        {
            "topic": "topic1",
            "partition": 8         
        },
        {
            "topic": "topic2",
            "partition": 16        
        }
     ]            
}
```
/admin/reassign_partitions用于将一些Partition分配到不同的broker集合上。 对于每个待重新分配的Partition，Kafka会在该znode上存储其所有的Replica和相应的Broker id。该znode由管理进程创建并且一旦重新分配成功它将会被自动移除。其数据结构如下：
```
{ 
"fields":[ 
{ 
"name":"version", 
"type":"int", 
"doc":"version id" 
}, 
{ 
"name":"partitions", 
"type":{ 
"type":"array", 
"items":{ 
"fields":[ 
{ 
"name":"topic", 
"type":"string", 
"doc":"topic of the partition to be reassigned" 
}, 
{ 
"name":"partition", 
"type":"int", 
"doc":"the partition to be reassigned" 
}, 
{ 
"name":"replicas", 
"type":"array", 
"items":"int", 
"doc":"a list of replica ids" 
} 
], 
} 
"doc":"an array of partitions to be reassigned to new replicas" 
} 
} 
] 
}
Example:
{
  "version": 1,
  "partitions":
     [
        {
            "topic": "topic3",
            "partition": 1,
            "replicas": [1, 2, 3]
        }
     ]            
}
```
/admin/delete_topics数据结构：
```
Schema:
{ "fields":
    [ {"name": "version", "type": "int", "doc": "version id"},
      {"name": "topics",
       "type": { "type": "array", "items": "string", "doc": "an array of topics to be deleted"}
      } ]
}

Example:
{
  "version": 1,
  "topics": ["topic4", "topic5"]
}
```
brokers
![kafka3](/public/image/kafka-3.png)
broker（即/brokers/ids/[brokerId]）存储“活着”的broker信息。数据结构如下：
```
Schema:
{ "fields":
    [ {"name": "version", "type": "int", "doc": "version id"},
      {"name": "host", "type": "string", "doc": "ip address or host name of the broker"},
      {"name": "port", "type": "int", "doc": "port of the broker"},
      {"name": "jmx_port", "type": "int", "doc": "port for jmx"}
    ]
}

Example:
{
    "jmx_port":-1,
    "host":"node1",
    "version":1,
    "port":9092
}
```
topic注册信息（/brokers/topics/[topic]），存储该topic的所有partition的 所有replica所在的broker id，第一个replica即为preferred replica，对一个给定的partition，它在同一个broker上最多只有一个replica,因此broker id可作为replica id。数据结构如下：
Schema:
```
{ "fields" :
    [ {"name": "version", "type": "int", "doc": "version id"},
      {"name": "partitions",
       "type": {"type": "map",
                "values": {"type": "array", "items": "int", "doc": "a list of replica ids"},
                "doc": "a map from partition id to replica list"},
      }
    ]
}
Example:
{
    "version":1,
    "partitions":
        {"12":[6],
        "8":[2],
        "4":[6],
        "11":[5],
        "9":[3],
        "5":[7],
        "10":[4],
        "6":[8],
        "1":[3],
        "0":[2],
        "2":[4],
        "7":[1],
        "3":[5]}
}

partition state（/brokers/topics/[topic]/partitions/[partitionId]/state） 结构如下：
Schema:
{ "fields":
    [ {"name": "version", "type": "int", "doc": "version id"},
      {"name": "isr",
       "type": {"type": "array",
                "items": "int",
                "doc": "an array of the id of replicas in isr"}
      },
      {"name": "leader", "type": "int", "doc": "id of the leader replica"},
      {"name": "controller_epoch", "type": "int", "doc": "epoch of the controller that last updated the leader and isr info"},
      {"name": "leader_epoch", "type": "int", "doc": "epoch of the leader"}
    ]
}

Example:
{
    "controller_epoch":29,
    "leader":2,
    "version":1,
    "leader_epoch":48,
    "isr":[2]
}

controller
/controller -> int (broker id of the controller)存储当前controller的信息
Schema:
{ "fields":
    [ {"name": "version", "type": "int", "doc": "version id"},
      {"name": "brokerid", "type": "int", "doc": "broker id of the controller"}
    ]
}
Example:
{
    "version":1,
　　"brokerid":8
}
```
/controller_epoch -> int (epoch)直接以整数形式存储controller epoch，而非像其它znode一样以JSON字符串形式存储。
## broker failover过程简介
Controller在ZooKeeper注册Watch，一旦有Broker宕机（这是用宕机代表任何让系统认为其die的情景，包括但不 限于机器断电，网络不可用，GC导致的Stop The World，进程crash等），其在ZooKeeper对应的znode会自动被删除，ZooKeeper会fire Controller注册的watch，Controller读取最新的幸存的Broker。
Controller决定set_p，该集合包含了宕机的所有Broker上的所有Partition。
对set_p中的每一个Partition
- 3.1 从/brokers/topics/[topic]/partitions/[partition]/state读取该Partition当前的ISR
- 3.2 决定该Partition的新Leader。如果当前ISR中有至少一个Replica还幸存，则选择其中一个作为新Leader，新的ISR则包含当前 ISR中所有幸存的Replica。否则选择该Partition中任意一个幸存的Replica作为新的Leader以及ISR（该场景下可能会有潜在 的数据丢失）。如果该Partition的所有Replica都宕机了，则将新的Leader设置为-1。
- 3.3 将新的Leader，ISR和新的leader_epoch及controller_epoch写入/brokers/topics/[topic]/partitions/[partition]/state。注意，该操作只有其version在3.1至3.3的过程中无变化时才会执行，否则跳转到3.1
直接通过RPC向set_p相关的Broker发送LeaderAndISRRequest命令。Controller可以在一个RPC操作中发送多个命令从而提高效率。
broker failover顺序图如下所示。
Kafka是由LinkedIn开发的一个分布式的消息系统，使用Scala编写，它以可水平扩展和高吞吐率而被广泛使用。目前越来越多的开源分布式处理系统如Cloudera、Apache Storm、Spark都支持与Kafka集成。InfoQ一直在紧密关注Kafka的应用以及发展，“Kafka剖析”专栏将会从架构设计、实现、应用场景、性能等方面深度解析Kafka。
本文在上篇文章基础上，更加深入讲解了Kafka的HA机制，主要阐述了HA相关各种场景，如Broker failover、Controller failover、Topic创建/删除、Broker启动、Follower从Leader fetch数据等详细处理过程。同时介绍了Kafka提供的与Replication相关的工具，如重新分配Partition等。
# Broker Failover过程
## Controller对Broker failure的处理过程
Controller在ZooKeeper的/brokers/ids节点上注册Watch。一旦有Broker宕 机（本文用宕机代表任何让Kafka认为其Broker die的情景，包括但不限于机器断电，网络不可用，GC导致的Stop The World，进程crash等），其在ZooKeeper对应的Znode会自动被删除，ZooKeeper会fire Controller注册的Watch，Controller即可获取最新的幸存的Broker列表。
Controller决定set_p，该集合包含了宕机的所有Broker上的所有Partition。
对set_p中的每一个Partition：
-  3.1 从/brokers/topics/[topic]/partitions/[partition]/state读取该Partition当前的ISR。
-  3.2 决定该Partition的新Leader。如果当前ISR中有至少一个Replica还幸存，则选择其中一个作为新Leader，新的ISR则包含当前 ISR中所有幸存的Replica。否则选择该Partition中任意一个幸存的Replica作为新的Leader以及ISR（该场景下可能会有潜在 的数据丢失）。如果该Partition的所有Replica都宕机了，则将新的Leader设置为-1。
-  3.3 将新的Leader，ISR和新的leader_epoch及controller_epoch写入/brokers/topics/[topic]/partitions/[partition]/state。注意，该操作只有Controller版本在3.1至3.3的过程中无变化时才会执行，否则跳转到3.1。
直接通过RPC向set_p相关的Broker发送LeaderAndISRRequest命令。Controller可以在一个RPC操作中发送多个命令从而提高效率。
Broker failover顺序图如下所示。
![kafka4](/public/image/kafka-4.png)
LeaderAndIsrRequest结构如下
![kafka5](/public/image/kafka-5.png)
LeaderAndIsrResponse结构如下
![kafka6](/public/image/kafka-6.png)
## 创建/删除Topic
Controller在ZooKeeper的/brokers/topics节点上注册Watch，一旦某个Topic被创建或删除，则Controller会通过Watch得到新创建/删除的Topic的Partition/Replica分配。
对于删除Topic操作，Topic工具会将该Topic名字存于/admin/delete_topics。若delete.topic.enable为true，则Controller注册在/admin/delete_topics上的Watch被fire，Controller通过回调向对应的Broker发送StopReplicaRequest，若为false则Controller不会在/admin/delete_topics上注册Watch，也就不会对该事件作出反应。
对于创建Topic操作，Controller从/brokers/ids读取当前所有可用的Broker列表，对于set_p中的每一个Partition：
- 3.1 从分配给该Partition的所有Replica（称为AR）中任选一个可用的Broker作为新的Leader，并将AR设置为新的ISR（因为该 Topic是新创建的，所以AR中所有的Replica都没有数据，可认为它们都是同步的，也即都在ISR中，任意一个Replica都可作为 Leader）
- 3.2 将新的Leader和ISR写入/brokers/topics/[topic]/partitions/[partition]
直接通过RPC向相关的Broker发送LeaderAndISRRequest。
创建Topic顺序图如下所示。
![kafka7](/public/image/kafka-7.png)
## Broker响应请求流程
Broker通过kafka.network.SocketServer及相关模块接受各种请求并作出响应。整个网络通信模块基于Java NIO开发，并采用Reactor模式，其中包含1个Acceptor负责接受客户请求，N个Processor负责读写数据，M个Handler处理业务逻辑。
Acceptor的主要职责是监听并接受客户端（请求发起方，包括但不限于Producer，Consumer，Controller，Admin Tool）的连接请求，并建立和客户端的数据传输通道，然后为该客户端指定一个Processor，至此它对该客户端该次请求的任务就结束了，它可以去响 应下一个客户端的连接请求了。其核心代码如下。
![kafka8](/public/image/kafka-8.png)
Processor主要负责从客户端读取数据并将响应返回给客户端，它本身并不处理具体的业务逻辑，并且其内部维护了一个队列来保存分配给它的所有SocketChannel。Processor的run方法会循环从队列中取出新的SocketChannel并将其SelectionKey.OP_READ注册到selector上，然后循环处理已就绪的读（请求）和写（响应）。Processor读取完数据后，将其封装成Request对象并将其交给RequestChannel。
RequestChannel是Processor和KafkaRequestHandler交换数据的地方，它包含一个队列 requestQueue用来存放Processor加入的Request，KafkaRequestHandler会从里面取出Request来处理； 同时它还包含一个respondQueue，用来存放KafkaRequestHandler处理完Request后返还给客户端的Response。
Processor会通过processNewResponses方法依次将requestChannel中responseQueue保存的Response取出，并将对应的SelectionKey.OP_WRITE事件注册到selector上。当selector的select方法返回时，对检测到的可写通道，调用write方法将Response返回给客户端。
KafkaRequestHandler循环从RequestChannel中取Request并交给kafka.server.KafkaApis处理具体的业务逻辑。
## LeaderAndIsrRequest响应过程
对于收到的LeaderAndIsrRequest，Broker主要通过ReplicaManager的becomeLeaderOrFollower处理，流程如下：
若请求中controllerEpoch小于当前最新的controllerEpoch，则直接返回ErrorMapping.StaleControllerEpochCode。
对于请求中partitionStateInfos中的每一个元素，即（(topic, partitionId), partitionStateInfo)：
- 2.1 若partitionStateInfo中的leader epoch大于当前ReplicManager中存储的(topic, partitionId)对应的partition的leader epoch，则：
- 2.1.1 若当前brokerid（或者说replica id）在partitionStateInfo中，则将该partition及partitionStateInfo存入一个名为partitionState的HashMap中
- 2.1.2 否则说明该Broker不在该Partition分配的Replica list中，将该信息记录于log中
*  2.2 否则将相应的Error code（ErrorMapping.StaleLeaderEpochCode）存入Response中
筛选出partitionState中Leader与当前Broker ID相等的所有记录存入partitionsTobeLeader中，其它记录存入partitionsToBeFollower中。
若partitionsTobeLeader不为空，则对其执行makeLeaders方。
若partitionsToBeFollower不为空，则对其执行makeFollowers方法。
若highwatermak线程还未启动，则将其启动，并将hwThreadInitialized设为true。
关闭所有Idle状态的Fetcher。
LeaderAndIsrRequest处理过程如下图所示
![kafka9](/public/image/kafka-9.png)
## Broker启动过程
Broker启动后首先根据其ID在ZooKeeper的/brokers/idszonde下创建临时子节点（Ephemeral node），创建成功后Controller的ReplicaStateMachine注册其上的Broker Change Watch会被fire，从而通过回调KafkaController.onBrokerStartup方法完成以下步骤：
向所有新启动的Broker发送UpdateMetadataRequest，其定义如下。
![kafka10](/public/image/kafka-10.png)
将新启动的Broker上的所有Replica设置为OnlineReplica状态，同时这些Broker会为这些Partition启动high watermark线程。
通过partitionStateMachine触发OnlinePartitionStateChange。
## Controller Failover
Controller也需要Failover。每个Broker都会在Controller Path (/controller) 上注册一个Watch。当前Controller失败时，对应的Controller Path会自动消失（因为它是Ephemeral Node），此时该Watch被fire，所有“活”着的Broker都会去竞选成为新的Controller（创建新的Controller Path），但是只会有一个竞选成功（这点由ZooKeeper保证）。竞选成功者即为新的Leader，竞选失败者则重新在新的Controller Path上注册Watch。因为ZooKeeper的Watch是一次性的，被fire一次之后即失效，所以需要重新注册。
Broker成功竞选为新Controller后会触发KafkaController.onControllerFailover方法，并在该方法中完成如下操作：
读取并增加Controller Epoch。
在ReassignedPartitions Patch(/admin/reassign_partitions)上注册Watch。
在PreferredReplicaElection Path(/admin/preferred_replica_election)上注册Watch。
通过partitionStateMachine在Broker Topics Patch(/brokers/topics)上注册Watch。
若delete.topic.enable设置为true（默认值是false），则partitionStateMachine在Delete Topic Patch(/admin/delete_topics)上注册Watch。
通过replicaStateMachine在Broker Ids Patch(/brokers/ids)上注册Watch。
初始化ControllerContext对象，设置当前所有Topic，“活”着的Broker列表，所有Partition的Leader及ISR等。
启动replicaStateMachine和partitionStateMachine。
将brokerState状态设置为RunningAsController。
将每个Partition的Leadership信息发送给所有“活”着的Broker。
若auto.leader.rebalance.enable配置为true（默认值是true），则启动partition-rebalance线程。
若delete.topic.enable设置为true且Delete Topic Patch(/admin/delete_topics)中有值，则删除相应的Topic。
## Partition重新分配
管理工具发出重新分配Partition请求后，会将相应信息写到/admin/reassign_partitions上，而该操作会触发ReassignedPartitionsIsrChangeListener，从而通过执行回调函数KafkaController.onPartitionReassignment来完成以下操作：
将ZooKeeper中的AR（Current Assigned Replicas）更新为OAR（Original list of replicas for partition） + RAR（Reassigned replicas）。
强制更新ZooKeeper中的leader epoch，向AR中的每个Replica发送LeaderAndIsrRequest。
将RAR - OAR中的Replica设置为NewReplica状态。
等待直到RAR中所有的Replica都与其Leader同步。
将RAR中所有的Replica都设置为OnlineReplica状态。
将Cache中的AR设置为RAR。
若Leader不在RAR中，则从RAR中重新选举出一个新的Leader并发送LeaderAndIsrRequest。若新的Leader不是从RAR中选举而出，则还要增加ZooKeeper中的leader epoch。
将OAR - RAR中的所有Replica设置为OfflineReplica状态，该过程包含两部分。第一，将ZooKeeper上ISR中的OAR - RAR移除并向Leader发送LeaderAndIsrRequest从而通知这些Replica已经从ISR中移除；第二，向OAR - RAR中的Replica发送StopReplicaRequest从而停止不再分配给该Partition的Replica。
将OAR - RAR中的所有Replica设置为NonExistentReplica状态从而将其从磁盘上删除。
将ZooKeeper中的AR设置为RAR。
删除/admin/reassign_partition。
注意：最后一步才将ZooKeeper中的AR更新，因为这是唯一一个持久存储AR的地方，如果Controller在这一步之前crash，新的Controller仍然能够继续完成该过程。
以下是Partition重新分配的案例，OAR = ｛1，2，3｝，RAR = ｛4，5，6｝，Partition重新分配过程中ZooKeeper中的AR和Leader/ISR路径如下
```
AR leader/isr Sttep
{1,2,3} 1/{1,2,3} (initial state)
{1,2,3,4,5,6} 1/{1,2,3} (step 2)
{1,2,3,4,5,6} 1/{1,2,3,4,5,6} (step 4)
{1,2,3,4,5,6} 4/{1,2,3,4,5,6} (step 7)
{1,2,3,4,5,6} 4/{4,5,6} (step 8)
{4,5,6} 4/{4,5,6} (step 10)
```
## Follower从Leader Fetch数据
Follower通过向Leader发送FetchRequest获取消息，FetchRequest结构如下
![kafka11](/public/image/kafka-11.png)
从FetchRequest的结构可以看出，每个Fetch请求都要指定最大等待时间和最小获取字节数，以及由TopicAndPartition 和PartitionFetchInfo构成的Map。实际上，Follower从Leader数据和Consumer从Broker Fetch数据，都是通过FetchRequest请求完成，所以在FetchRequest结构中，其中一个字段是clientID，并且其默认值是 ConsumerConfig.DefaultClientId。
Leader收到Fetch请求后，Kafka通过KafkaApis.handleFetchRequest响应该请求，响应过程如下：
replicaManager根据请求读出数据存入dataRead中。
如果该请求来自Follower则更新其相应的LEO（log end offset）以及相应Partition的High Watermark
根据dataRead算出可读消息长度（单位为字节）并存入bytesReadable中。
满足下面4个条件中的1个，则立即将相应的数据返回
Fetch请求不希望等待，即fetchRequest.macWait <= 0
Fetch请求不要求一定能取到消息，即fetchRequest.numPartitions <= 0，也即requestInfo为空
有足够的数据可供返回，即bytesReadable >= fetchRequest.minBytes
读取数据时发生异常
若不满足以上4个条件，FetchRequest将不会立即返回，并将该请求封装成DelayedFetch。检查该DeplayedFetch是否满足，若满足则返回请求，否则将该请求加入Watch列表
Leader通过以FetchResponse的形式将消息返回给Follower，FetchResponse结构如下
![kafka12](/public/image/kafka-12.png)
Replication工具
```
Topic Tool
$KAFKA_HOME/bin/kafka-topics.sh，该工具可用于创建、删除、修改、查看某个Topic，也可用于列出所有Topic。另外，该工具还可修改某个Topic的以下配置。
unclean.leader.election.enable
delete.retention.ms
segment.jitter.ms
retention.ms
flush.ms
segment.bytes
flush.messages
segment.ms
retention.bytes
cleanup.policy
segment.index.bytes
min.cleanable.dirty.ratio
max.message.bytes
file.delete.delay.ms
min.insync.replicas
index.interval.bytes
Replica Verification Tool
$KAFKA_HOME/bin/kafka-replica-verification.sh，该工具用来验证所指定的一个或多个Topic下每个Partition对应的所有Replica是否都同步。可通过topic-white-list这一参数指定所需要验证的所有Topic，支持正则表达式。
Preferred Replica Leader Election Tool
```
用途
有了Replication机制后，每个Partition可能有多个备份。某个Partition的Replica列表叫作 AR（Assigned Replicas），AR中的第一个Replica即为“Preferred Replica”。创建一个新的Topic或者给已有Topic增加Partition时，Kafka保证Preferred Replica被均匀分布到集群中的所有Broker上。理想情况下，Preferred Replica会被选为Leader。以上两点保证了所有Partition的Leader被均匀分布到了集群当中，这一点非常重要，因为所有的读写操作 都由Leader完成，若Leader分布过于集中，会造成集群负载不均衡。但是，随着集群的运行，该平衡可能会因为Broker的宕机而被打破，该工具 就是用来帮助恢复Leader分配的平衡。
事实上，每个Topic从失败中恢复过来后，它默认会被设置为Follower角色，除非某个Partition的Replica全部宕机，而当前 Broker是该Partition的AR中第一个恢复回来的Replica。因此，某个Partition的Leader（Preferred Replica）宕机并恢复后，它很可能不再是该Partition的Leader，但仍然是Preferred Replica。
原理
1. 在ZooKeeper上创建/admin/preferred_replica_election节点，并存入需要调整Preferred Replica的Partition信息。
2. Controller一直Watch该节点，一旦该节点被创建，Controller会收到通知，并获取该内容。
3. Controller读取Preferred Replica，如果发现该Replica当前并非是Leader并且它在该Partition的ISR中，Controller向该Replica发送 LeaderAndIsrRequest，使该Replica成为Leader。如果该Replica当前并非是Leader，且不在ISR 中，Controller为了保证没有数据丢失，并不会将其设置为Leader。
用法
```
$KAFKA_HOME/bin/kafka-preferred-replica-election.sh --zookeeper localhost:2181
```
在包含8个Broker的Kafka集群上，创建1个名为topic1，replication-factor为3，Partition数为8的Topic，使用如下命令查看其Partition/Replica分布。
```
$KAFKA_HOME/bin/kafka-topics.sh --describe --topic topic1 --zookeeper localhost:2181
```
查询结果如下图所示，从图中可以看到，Kafka将所有Replica均匀分布到了整个集群，并且Leader也均匀分布。
![kafka12](/public/image/kafka-12.png)
手动停止部分Broker，topic1的Partition/Replica分布如下图所示。从图中可以看到，由于Broker 1/2/4都被停止，Partition 0的Leader由原来的1变为3，Partition 1的Leader由原来的2变为5，Partition 2的Leader由原来的3变为6，Partition 3的Leader由原来的4变为7。
![kafka13](/public/image/kafka-13.png)
再重新启动ID为1的Broker，topic1的Partition/Replica分布如下。可以看到，虽然Broker 1已经启动（Partition 0和Partition5的ISR中有1），但是1并不是任何一个Parititon的Leader，而Broker 5/6/7都是2个Partition的Leader，即Leader的分布不均衡——一个Broker最多是2个Partition的Leader，而 最少是0个Partition的Leader。
![kafka14](/public/image/kafka-14.png)
运行该工具后，topic1的Partition/Replica分布如下图所示。由图可见，除了Partition 1和Partition 3由于Broker 2和Broker 4还未启动，所以其Leader不是其Preferred Repliac外，其它所有Partition的Leader都是其Preferred Replica。同时，与运行该工具前相比，Leader的分配更均匀——一个Broker最多是2个Parittion的Leader，最少是1个 Partition的Leader。
![kafka15](/public/image/kafka-15.png)
启动Broker 2和Broker 4，Leader分布与上一步相比并未变化，如下图所示。
![kafka16](/public/image/kafka-16.png)
再次运行该工具，所有Partition的Leader都由其Preferred Replica承担，Leader分布更均匀——每个Broker承担1个Partition的Leader角色。
除了手动运行该工具使Leader分配均匀外，Kafka还提供了自动平衡Leader分配的功能，该功能可通过将auto.leader.rebalance.enable设置为true开启，它将周期性检查Leader分配是否平衡，若不平衡度超过一定阈值则自动由Controller尝试将各Partition的Leader设置为其Preferred Replica。检查周期由leader.imbalance.check.interval.seconds指定，不平衡度阈值由leader.imbalance.per.broker.percentage指定。
## Kafka Reassign Partitions Tool
用途
该工具的设计目标与Preferred Replica Leader Election Tool有些类似，都旨在促进Kafka集群的负载均衡。不同的是，Preferred Replica Leader Election只能在Partition的AR范围内调整其Leader，使Leader分布均匀，而该工具还可以调整Partition的AR。
Follower需要从Leader Fetch数据以保持与Leader同步，所以仅仅保持Leader分布的平衡对整个集群的负载均衡来说是不够的。另外，生产环境下，随着负载的增大，可 能需要给Kafka集群扩容。向Kafka集群中增加Broker非常简单方便，但是对于已有的Topic，并不会自动将其Partition迁移到新加 入的Broker上，此时可用该工具达到此目的。某些场景下，实际负载可能远小于最初预期负载，此时可用该工具将分布在整个集群上的Partition重 装分配到某些机器上，然后可以停止不需要的Broker从而实现节约资源的目的。
需要说明的是，该工具不仅可以调整Partition的AR位置，还可调整其AR数量，即改变该Topic的replication factor。
原理
该工具只负责将所需信息存入ZooKeeper中相应节点，然后退出，不负责相关的具体操作，所有调整都由Controller完成。
- 1. 在ZooKeeper上创建/admin/reassign_partitions节点，并存入目标Partition列表及其对应的目标AR列表。
- 2. Controller注册在/admin/reassign_partitions上的Watch被fire，Controller获取该列表。
- 3. 对列表中的所有Partition，Controller会做如下操作：
启动RAR - AR中的Replica，即新分配的Replica。（RAR = Reassigned Replicas， AR = Assigned Replicas）
等待新的Replica与Leader同步
如果Leader不在RAR中，从RAR中选出新的Leader
停止并删除AR - RAR中的Replica，即不再需要的Replica
删除/admin/reassign_partitions节点
用法
该工具有三种使用模式
generate模式，给定需要重新分配的Topic，自动生成reassign plan（并不执行）
execute模式，根据指定的reassign plan重新分配Partition
verify模式，验证重新分配Partition是否成功
下面这个例子将使用该工具将Topic的所有Partition重新分配到Broker 4/5/6/7上，步骤如下：
- 1. 使用generate模式，生成reassign plan
指定需要重新分配的Topic （{"topics":[{"topic":"topic1"}],"version":1}），并存入/tmp/topics-to-move.json文件中，然后执行如下命令
$KAFKA_HOME/bin/kafka-reassign-partitions.sh --zookeeper localhost:2181
--topics-to-move-json-file /tmp/topics-to-move.json 
--broker-list "4,5,6,7" --generate

结果如下图所示
![kafka17](/public/image/kafka-17.png)
- 2. 使用execute模式，执行reassign plan
将上一步生成的reassignment plan存入/tmp/reassign-plan.json文件中，并执行
```$KAFKA_HOME/bin/kafka-reassign-partitions.sh --zookeeper localhost:2181 
--reassignment-json-file /tmp/reassign-plan.json --execute
```

此时，ZooKeeper上/admin/reassign_partitions节点被创建，且其值与/tmp/reassign-plan.json文件的内容一致。

- 3. 使用verify模式，验证reassign是否完成
执行verify命令
```$KAFKA_HOME/bin/kafka-reassign-partitions.sh --zookeeper localhost:2181 
--reassignment-json-file /tmp/reassign-plan.json --verify
```
结果如下所示，从图中可看出topic1的所有Partititon都根据reassign plan重新分配成功。
![kafka18](/public/image/kafka-18.png)
接下来用Topic Tool再次验证。
```bin/kafka-topics.sh --zookeeper localhost:2181 --describe --topic topic1
```
结果如下图所示，从图中可看出topic1的所有Partition都被重新分配到Broker 4/5/6/7，且每个Partition的AR与reassign plan一致。
![kafka19](/public/image/kafka-19.png)
需要说明的是，在使用execute之前，并不一定要使用generate模式自动生成reassign plan，使用generate模式只是为了方便。事实上，某些场景下，generate模式生成的reassign plan并不一定能满足需求，此时用户可以自己设置reassign plan。
State Change Log Merge Tool
用途
该工具旨在从整个集群的Broker上收集状态改变日志，并生成一个集中的格式化的日志以帮助诊断状态改变相关的故障。每个Broker都会将其收到的状态改变相关的的指令存于名为state-change.log的日志文件中。某些情况下，Partition的Leader election可能会出现问题，此时我们需要对整个集群的状态改变有个全局的了解从而诊断故障并解决问题。该工具将集群中相关的state-change.log日志按时间顺序合并，同时支持用户输入时间范围和目标Topic及Partition作为过滤条件，最终将格式化的结果输出。
用法
```bin/kafka-run-class.sh kafka.tools.StateChangeLogMerger 
--logs /opt/kafka_2.11-0.8.2.1/logs/state-change.log 
--topic topic1 --partitions 0,1,2,3,4,5,6,7
```
