Redis集群是Redis提供的分布式数据库方法，集群通过分片（sharding）来进行数据共享，并提供复制和故障转移功能。

## 节点

一个Redis集群通常由多个节点（node）组成，在刚开始的时候，每个节点都是互相独立的，它们都处于一个只包含自己的集群中。然后客户端可以通过CLUSTER MEET命令让节点与指定的节点握手，握手成功后新节点就会添加到当前节点所在的集群中。

#### 启动节点

一个节点就是一个运行在集群模式下的Redis服务器，Redis服务器在启动时根据cluster-enabled配置选项是否为yes来决定是否开启服务器的集群模式。

![判断是否开启集群模式](https://github.com/codzeroNov/MyNotes/blob/master/Redis/PICS/%E5%88%A4%E6%96%AD%E6%98%AF%E5%90%A6%E5%BC%80%E5%90%AF%E9%9B%86%E7%BE%A4%E6%A8%A1%E5%BC%8F.png)

#### 集群的数据结构

节点clusterNode结构如下：

```c
typedef struct clusterNode {
    //  创建节点时间
    mstime_t ctime;
    //  节点的名字，40个十六进制字符组成
    char name[REDISCLUSTER_NAMELEN];
    //  节点标识
    //  使用各种不同的标识值记录节点的角色（比如：主节点、从节点）
    //  以及节点目前所处的状态（比如：在线或者下线）
    int flags;
    
    //  节点当前的配置纪元，用于实现故障转移
    uint64_t configEpoch;
    
    //  记录节点的槽指派信息,CLUSTER_SLOTS=16384
    unsigned char slots[CLUSTER_SLOTS/8];
    
    //  记录节点负责处理的槽的数量
    int numslots;
    
    //  主节点的slave节点数量
    int numslaves;
    
    //  从节点信息
    struct clusterNode **slaves;
    
    //  节点的ip地址
    char ip[NET_IP_STR_LEN];
    
    //  节点的端口号
    int port;
    
    //  保存连接节点所需的有关信息
    clusterLink *link; 
    //  ……
} clusterNode;
```

其中，link属性是一个clusterLink结构。该结构保存了连接节点所需的相关信息。结构如下：

```c
typedef struct clusterLink {

    // 连接的创建时间
    mstime_t ctime;
    
    // TCP套接字描述符
    int fd;
    
    // 输入缓冲区，保存着等待发送给其他节点的消息(message)
    sds sndbuf;
    
    // 与这个连接相关联的节点，如果没有的话就为NULL
    struct clusterNode *node;
}
```

*注：redisClient结构和clusterLink结构都有自己的套接字描述符和输入、输出缓冲区，不同在于，一个是用来连接客户端的，一个是用于连接节点的。*

每个节点都保存着一个clusterState结构，记录当前节点的视角下，集群的情况。结构如下：

```c
typedef struct clusterState {
    
    //指向当前节点的指针
    clusterNode *myself;
    
    //节点当前的配置纪元，用于实现故障转移
    uint64_t currentEpoch;
    
    //集群当前的状态：是线上还是下线
    int state;
    
    //集群中至少处理着一个槽的节点数量
    int size;
    
    //集群节点名单（包括myself节点）
    //字典的键为节点的名字，字典的值为节点对应的clusterNode结构
    dict *nodes;
    
    //...
} clusterState;
```

下图表示7000节点视角的clusterState结构：

![7000节点视角的clusterState结构](https://github.com/codzeroNov/MyNotes/blob/master/Redis/PICS/7000%E8%8A%82%E7%82%B9%E8%A7%86%E8%A7%92%E7%9A%84clusterState%E7%BB%93%E6%9E%84.png)

#### CLUSTER MEET命令的实现

通过向节点A发送CLUSTER MEET命令，客户端可以让接收命令的节点A将另一个节点B添加到节点A当前所在的集群里：`CLUSTER MEET <ip> <port>`

![节点的握手过程](https://github.com/codzeroNov/MyNotes/blob/master/Redis/PICS/%E8%8A%82%E7%82%B9%E7%9A%84%E6%8F%A1%E6%89%8B%E8%BF%87%E7%A8%8B.png)

之后，节点A会将节点B的信息通过Gossip协议传播给集群中的其他节点，让其他节点与节点B进行握手。

> Gossip协议属于最终一致性协议。

## 槽指派

Redis集群通过分片的方式来保存数据库中的键值对：集群的整个数据库被分为16384个槽（slot），数据库中的每个键都属于这16384个槽中的一个，集群中每个节点可以处理0个或最多16384个槽。

当数据库中的16384个槽都有节点在处理时，集群处于上线状态（ok）；否则，如有任何一个槽没有得到处理，集群处于下线状态（fail）。

#### 记录节点的槽指派信息

clusterNode结构的slots属性和numslot属性记录了节点负责处理哪些槽：

```c
struct clusterNode {
    //...
    
    //  记录节点的槽指派信息,CLUSTER_SLOTS=16384
    unsigned char slots[CLUSTER_SLOTS/8];
    
    //  记录节点负责处理的槽的数量
    int numslots;
   
    //... 
}
```

slots属性是一个二进制位数组（bit array），长度为16384/8=2048个字节。如果所以i上的二进制位为1，那么节点负责处理槽i。为0则不处理。

#### 传播节点的槽指派信息

节点除了会将自己负责处理的槽记录在clusterNode结构的slots属性和numslots属性之外，它还会将自己的slots数组通过消息发送给集群中的其他节点，以此来告知其他节点自己目前负责处理哪些槽。

所以，集群中每个节点都会知道数据库中16384个槽分别被指派给了集群中哪些节点。

#### 记录集群所有槽的指派信息

clusterState结构中的slots数组记录了集群中所有16384个槽的指派信息：

```shell
typedef struct clusterState {
    clusterNode *slots[16384];   
}
```

![slots数组](https://github.com/codzeroNov/MyNotes/blob/master/Redis/PICS/slots%E6%95%B0%E7%BB%84.png)

clusterNode.slots用来记录节点负责哪些槽；clusterState.slots用来记录每个槽是哪些节点负责。这么做是为了加速定位。

#### CLUSTER ADDSLOTS命令实现

CLUSTER ADDSLOTS命令接受一个或多个槽作为参数，并将所有输入的槽指派给接收该命令的节点负责。

![节点的CLUSTER STATE结构](https://github.com/codzeroNov/MyNotes/blob/master/Redis/PICS/%E8%8A%82%E7%82%B9%E7%9A%84CLUSTER%20STATE%E7%BB%93%E6%9E%84.png)

## 在集群中执行命令

对所有槽进行指派后，集群就会进入上线状态，这时的客户端就可以向集群中的节点发送数据命令。

当客户端向节点发送与数据库键有关的命令时，接收命令的节点会计算出命令要处理的数据库键属于哪个槽，并检查这个槽是否指派给了自己：

1. 若该槽指派给自己，则执行命令。
2. 槽没有指派给当前节点，则返回MOVED错误并重定向（redirect）到正确的节点。

![判断客户端是否需要转向的流程](https://github.com/codzeroNov/MyNotes/blob/master/Redis/PICS/%E5%88%A4%E6%96%AD%E5%AE%A2%E6%88%B7%E7%AB%AF%E6%98%AF%E5%90%A6%E9%9C%80%E8%A6%81%E8%BD%AC%E5%90%91%E7%9A%84%E6%B5%81%E7%A8%8B.png)

#### 节点数据库的实现

集群节点保存键值对以及键值对过期时间的方式与单点时一致。

集群节点只能使用0号数据库，而单点没有该限制。

集群节点还会使用clusterState结构中的slots_to_keys跳跃表来保存槽和键之间的关系：

```c
typedef struct clusterState {
    zskiplist *slots_to_keys;   
}
```

跳跃表的每个分值（score）都是一个槽号，每个节点的成员（member）都是一个数据库键：

![slots_to_keys跳跃表](https://github.com/codzeroNov/MyNotes/blob/master/Redis/PICS/slots_to_keys%E8%B7%B3%E8%B7%83%E8%A1%A8.png)

## 重新分片

重新分片可以将任意数量已经指派给某个节点的槽改为指派给另一个节点，并且槽所属的键值对也会相应移动到目标节点。重新分片可以在线进行，集群不需要下线，并且源节点和目标节点可以继续处理请求。

#### 重新分片的实现原理

redis-trib负责集群的重新分片操作。

1. redis-trib对目标节点发送 ` CLUSTER SETSLOT <slot> IMPORTING <source_id> `命令，让目标节点准备好从源节点导入属于槽slot的键值对。

2. redis-trib对源节点发送` CLUSTER SETSLOT <slot> MIGRATING <target_id> `命令，让源节点准备好将属于槽的键值对迁移到目标节点。

3. redis-trib向源节点发送` CLUSTER GETKEYSINSLOT <slot> <count> `命令，获得最多count个属于槽slot的键值对的键名。

4. 对步骤3获得的每个键名，redis-trib向源节点发送一个` MIGRATE <target_id> <target_port> <key_name> 0 <timeout> ` 命令，将被选中的键原子地从源节点迁移至目标节点。

5. 重复3、4，至全部完成迁移。

6. redis-trib向集群中的任意一个节点发送``` CLUSTER SETSLOT <slot> NODE <target_id>```命令，让所有节点知道这一变化。

键迁移过程：

![迁移键的过程](https://github.com/codzeroNov/MyNotes/blob/master/Redis/PICS/%E8%BF%81%E7%A7%BB%E9%94%AE%E7%9A%84%E8%BF%87%E7%A8%8B.png)

槽重新分片过程：

![对槽进行重新分片的过程](D:\DOCS\REDIS\PICS\对槽进行重新分片的过程.png)

## ASK错误

在重新分片过程中，可能有这样的情况：属于被迁移槽的一部分在源节点里，另一部分在目标节点里。

当客户端向源节点发送一个与数据库键有关的命令时，并且命令要处理的数据库键恰好就属于正在被迁移的槽时：

+ 源节点会先在自己的数据库里面查找指定的键，如存在则执行客户端命令。
+ 如未找到，则源节点向客户端返回ASK错误，指引客户端转向正在导入槽的目标节点，并再次发送之前想要执行的命令。

![判断是否发送ASK错误的过程](D:\DOCS\REDIS\PICS\判断是否发送ASK错误的过程.png)

> 与MOVED错误类似，集群模式下的客户端接到ASK错误时也不会打印，而是自动根据错误提供的IP地址和端口进行转向操作。
>
> + MOVED错误代表槽的负责全已经从一个节点转移到了另一个节点：在客户端收到关于槽i的MOVED错误之后，客户端每次遇到关于槽i的命令请求时，都可以直接将命令请求发送至MOVED错误所指向的节点，因为该节点就是目前负责槽i的节点。
> + ASK错误只是两个节点在迁移槽的过程中使用的临时措施：客户端收到关于槽i的ASK错误后，客户端只会在下一次请求发送到ASK错误指向的节点，但这种转向不会对客户端今后发送关于槽i的请求产生任何影响，客户端仍然会将关于槽i的命令请求发送至目前负责处理槽i的节点。

## 复制与故障转移

Redis集群中分为主节点和从节点，主节点用于处理槽，从节点用于复制主节点，并在被复制的主节点下线时，代替主节点继续处理命令请求。

#### 设置从节点

向一个节点发送`CLUSTER REPLICATE <node_id>`可以让接收命令的节点成为node_id所指定节点的从节点。

该从节点正在复制某一主节点的信息会发送给集群中的其他节点。集群中的所有节点都会在代表主节点的clusterNode结构的slaves属性中记录正在复制这个主节点的从节点名单：

```c
struct clusterNode {
    //正在复制这个主节点的从节点数量
    int numslaves;

    //一个数组，指向一个正在复制这个主节点的从节点的clusterNode结构
    struct clusterNode **slaves;

}
```

#### 故障检测

集群中的每个节点都会定期地向其他节点发送PING消息，以此来检测对方是否在线，如果接收PING消息的节点没在规定时间内相应，那么发送PING消息的节点就会将其标记为疑似下线（probable fail，PFAIL）。

当一个主节点A通过消息得知主节点B认为主节点C进入了疑似下线状态时，主节点A会在自己的clusterState.nodes字典中找到主节点C所对应的clusterNode结构，并将主节点B的下线报告（failure report）添加到clusterNode结构的fail_reports链表里。

![节点7000的下线报告](D:\DOCS\REDIS\PICS\节点7000的下线报告.png)

如果在一个集群里，半数以上的主节点都将某个主节点x报告为疑似下线，那么这个节点x将被标记为已下线（FAIL），然后向集群广播一条关于节点x已下线的FAIL消息，所有收到该消息的节点都会立即标注x为已下线。

#### 故障转移

当一个从节点发现自己复制的主节点进入已下线状态时，从节点将开始故障转移：

1. 已下线主节点的一个从节点被选出。
2. 被选出的从节点执行SLAVEOF no one命令，成为新主节点。
3. 新主节点会将已下线主节点的槽全都指派给自己。
4. 新主节点向集群广播一条PONG消息，告知集群中其他节点自己成为新主节点，并接管相应的槽。
5. 新主节点开始接收自己负责的槽的相关命令请求，故障转移完成。

#### 选举新的主节点

以下时集群选举新主节点的方法：

1. 集群的配置纪元是一个自增计数器，它的初始值为0.
2. 当集群的某个节点开始一次故障转移时，集群的配置纪元会增一。
3. 对于每个配置纪元，集群里每个负责处理槽的主节点都有一次投票的机会，而第一个向主节点要求投票的从节点将获得主节点的投票。
4. 从节点发现自己正在复制的主节点进入已下线状态时，会向集群广播一条`CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST`（REQUEST）消息，要求所有收到该消息且有投票权的主节点向该从节点投票。
5. 如果一个主节点具有投票权，并且这个主节点尚未投票给其他从节点，那么主节点将向要求投票的从节点返回一条`CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK`（ACK）消息，表示这个主节点支持从节点成为新节点。
6. 每个参与选举的从节点都会收到ACK消息，并统计自己获得了多少支持。若获得半数以上的支持，则成为新主节点。
7. 如果一个纪元里没有新主节点选出，则进入下一纪元，重新选举。

> 新主节点选举和Sentine选举类似，都是基于Raft实现。

## 消息

集群中发送消息的节点成为发送者（sender），接收消息的节点称为接收者（receiver）。

+ MEET消息：当发送者接收到客户端发送的CLUSTER MEET命令时，发送者会向接收者发送MEET消息，请求接收者加入到发送者当前的集群中。
+ PING消息：（1）集群中每个节点默认每隔1秒就会从已知节点列表中随机选出5个节点，对这5个节点中最长没有发送过PING消息的节点发送PING消息，以此检测节点是否在线。（2）如果节点A收到节点B的回复PONG超过cluster-node-timeout选项设置的一半，也会向节点B发送PING消息。
+ PONG消息：（1）接收者收到MEET或PING时使用PONG消息回复。（2）节点可以通过广播PONG消息来刷新集群其他节点对自己的认知，如故障转移成功后。
+ FAIL消息：当一个主节点A判断另一主节点B已经进入FAIL状态时，节点A会向集群广播一条关于节点B的FAIL消息，所有收到这条消息的节点都会立即将节点B标记为已下线。
+ PUBLISH消息：当节点接收到一个PUBLISH命令时，节点会执行这个命令，并向集群广播一条PUBLISH消息，所有接收到这条消息的节点都会执行同样的命令。

一条消息由消息头（header）和消息正文（data）组成。

#### 消息组成

结构：

```c
typedef struct {
    //消息的长度（包括消息的长度和正文的长度）
    uint32_t totlen;
    
    //消息的类型
    uint16_t type;
    
    //消息正文包含的节点信息数量（MEET、PING、PONG这三种Gossip协议消息时使用）
    uint16_t count;
    
    //发送者所处的配置纪元
    uint64_t currentEpoch;
    
    //主节点，那么记录的是发送者的配置纪元
    //从节点，那么记录的是发送者正在复制的主节点的配置纪元
    uint64_t configEpoch;
    
    //发送者的名字（ID）
    char sender[REDIS_CLUSTER_NAMELEN];
    
    //发送者目前的槽指派信息
    unsigned char myslots[REDIS_CLUSTER_SLOTS/8];
    
    //从节点，那么记录的是发送者正在复制的主节点命令
    //主节点，那么记录的是REDIS_NODE_NULL_NAME
    char slaveof[REDIS_CLUSTER_NAMELEN];
    
    //发送者的端口号
    uint16_t port;
    
    //发送者的标识值
    uint16_t flags;
    
    //发送者所处集群的状态
    unsigned char state;
    
    //消息的正文（或者说，内容）
    union clusterMsgData data;
}
```

clusterMsg.data即消息的正文：

```c
union clusterMsgData {
    //MEET、PING、PONG消息的正文
    struct {
        //每条MEET、PING、PONG消息都包含两个
        //clusterMsgDataGossip结构
        clusterMsgDataGossip gossip[1];
    } ping;
    
    //FALL消息的正文
    struct {
        clusterMsgDataFail about;
    } fail;
    
    //PUBLISH消息的正文
    struct {
        clusterMsgDataPublish msg;
    } publish;
    
    //其他消息的正文...
}
```

clusterMsg结构的currentEpoch、sender、myslots等属性记录了发送者自身的节点信息，接收者会根据这些信息，在自己的clusterState.nodes字典里找到发送者对应的clusterNode结构，并对结构进行更新。

#### MEET、PING、PONG消息的实现

Redis集群中各节点通过Gossip协议来交换各自关于不同节点的状态信息，其中Gossip协议由MEET、PING、PONG三种消息实现。

发送者每次发送该类消息时，都会从自己的已知节点（主或从）选出两个节点，并将这两个节点的信息b保存到两个clusterMsgDataGossip结构里。

```c
union clusterMsgData {

    //... 
    
    //MEET、PING和PONG消息的正文
    struct {
        //每条MEET、PING、PONG消息都包含两个
        //clusterMsgDataGossip结构
        clusterMsgDataGossip gossip[1];
    } ping;
    
    //其他消息的正文...
}
```

clusterMsgDataGossip结构如下：

```c
typedef struct {
   
    //节点的名字
    char nodename [REDIS_CLUSTER_NAMELEN];
    
    //最后一次向该节点发送PING消息的时间戳
    uint32_t ping_sent;
    
    //最后一次从该节点接到PONG消息的时间戳
    uint32_t pong_received;
    
    //节点的IP地址
    char ip[16];
    
    //节点的端口号
    uint16_t port;
    
    //节点的标识值
    uint16_t flags;
    
} clusterMsgDataGossip
```

接收者会根据自己是否认识这两个节点执行以下操作：

+ 不存在于接收者已知节点列表：接收者与这两个节点进行握手。
+ 存在于接收者已知节点列表：更新这两个节点对应的clusterNode结构。

#### FAIL消息的实现

当集群比较大时，采用Gossip协议来传播已下线信息延迟会比较高，所以FAIL消息采用向所有节点广播的形式实现。

#### PUBLISH消息的实现

向集群中某个节点发送`PUBLISH <channel> <message>`将导致集群中所有节点都向channel频道发送message消息。

![PUBLISH消息广播](D:\DOCS\REDIS\PICS\PUBLISH消息广播.png)

> Q：为什么不直接向节点广播PUBLISH命令？
>
> A：如果直接向各个节点广播PUBLISH命令，就违反了“各个节点通过发送和接收消息来进行通信”的规则。