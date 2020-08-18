哨兵是Redis的高可用性解决方案：由一个或多个Sentinel实例组成Sentinel系统，可以监视任意多个主服务器，以及这些主服务器的属下的从服务器，并在被监视的主服务器下线时，自动将下线的主服务器属下的某个从服务器升级为新的主服务器。

![服务器和Sentinel系统](D:\DOCS\REDIS\PICS\服务器和Sentinel系统.png)

![主服务器下线](D:\DOCS\REDIS\PICS\主服务器下线.png)

![故障转移](D:\DOCS\REDIS\PICS\故障转移.png)

![原主服务器降级](D:\DOCS\REDIS\PICS\原主服务器降级.png)

## 启动并初始化Sentinel

1. 初始化服务器
2. 将普通Redis服务器的代码替换成Sentinel专用代码
3. 初始化Sentinel状态
4. 根据给定的配置文件，初始化Sentinel监视的主服务器列表
5. 创建连向主服务器的网络连接

#### 初始化服务器

Sentinel本质上是一个运行在特殊模式下的Redis服务器，不使用数据库。

#### 使用Sentinel专用代码

将一部分普通Redis服务器使用的代码替换成Sentinel专用代码。

#### 初始化Sentinel状态

应用Sentinel的专用代码之后，服务器会初始化Sentinel状态的结构。这个结构保存了服务器中所有和Sentinel功能有关的状态。

#### 初始化Sentinel状态的masters属性

Sentinel状态的masters字典中记录了所有被Sentinel监视的主服务器的相关信息。其中：

+ 字典的键是被监视主服务器的名字
+ 字典的值代表了一个被Sentinel监视的Redis服务器实例（主、从或者另一个Sentinel）。

#### 创建连向主服务器的网络连接

Sentinel将成为主服务器的客户端，它可以向主服务器发送命令，并从命令回复中获取相关信息。Sentinel会创建两个连向主服务器的异步网络连接：

+ 命令连接。用于向主服务器发送命令，并接受命令回复。
+ 订阅连接。用于订阅主服务器的sentinel:hello频道

> 在Redis目前的发布与订阅功能中，被发送的信息都不会保存在Redis服务器里。如果在发送信息时，接受信息的客户端不在线或断线，那么这个客户端就会丢失该条信息。所以Sentinel创建sentinel:hello这个专用的订阅连接来接收该频道的信息。
>
> 另外，除了订阅频道外，Sentinel还必须向主服务器发送命令，所以需要命令连接。

## 获取主服务器信息

Sentinel默认会以每10秒一次的频率，向被监视的主服务器发送INFO命令，获取主服务器的信息：

+ 主服务器本身的信息。run_id、role等。
+ 主服务器属下从服务器的信息。

主服务器掉线重连后，run_id会改变，Sentinel会对主服务器的runid进行更新。

## 获取从服务器的信息

当Sentinel发现主服务器有新的从服务器出现时，Sentinel除了会为这个新的从服务器创建相应的实例结构外，还会创建连接到从服务器的命令连接和订阅连接。

创建命令连接后，Sentinel会默认以每10秒一次的频率通过命令连接向从服务器发送INFO命令。

## 向主服务器和从服务器发送信息

Sentinel默认会以每2秒一次通过命令连接向所有被监视的主和从服务器发送命令：

```
PUBLISH __sentinel__:hello "<s_ip>,<s_port>,<s_runid>,<s_epoch>,<m_name>,<m_ip>,<m_port>,<m_epoch>"
```

这条命令向服务器的sentinel:hello频道发送了一条消息。s开头的表示Sentinel本身的信息；m开头的表示主服务器的信息。

如果监视的是主服务器，参数记录的就是主服务器信息。如果监视的是从服务器，参数记录的就是从服务器正在复制的主服务器信息。

## 接收来自主服务器和从服务器的频道信息

Sentinel通过命令连接向服务器的sentinel:hello频道发送信息，又通过订阅连接从sentinel:hello频道接收信息。

对于监视同一个服务器的多个Sentine，一个Sentinel发送的信息会被其他Sentinel接收到。

![Sentinel向服务器发送信息](D:\DOCS\REDIS\PICS\Sentinel向服务器发送信息.png)

## 更新Sentinel字典

 Sentinel为主服务器创建的实例结构中的Sentinel字典除Sentinel本身之外，也保存同样监视这个服务器的其他Sentinel资料。

当一个Sentinel接收到其他Sentinel发来的消息时，目标Sentinel会从根据收到的消息，对其sentinels参数就行增改。

## 创建连向其他Sentinel的命令连接

当Sentinel通过频道信息发现一个新的Sentinel时，它不仅会为新的Sentinel在sentins字典中创建相应的实例结构，还会创建一个连向新Sentinel的命令连接，而新Sentinel也会创建连向这个Sentinel的命令连接。

![各个Sentinel之间的网络连接](D:\DOCS\REDIS\PICS\各个Sentinel之间的网络连接.png)

## 主观检测下线状态

默认情况下，Sentinel会以每秒一次的频率向所有与它创建了命令连接的实例（主、从服务器和其他Sentinel）发送PING命令，并通过PING命令回复来判断实例是否在线。

+ 有效回复：实例返回+PONG、-LOADING、-MASTERDOWN三种回复中的一种。
+ 无效回复：除有效回复之外的回复。

如果实例在down-after-milliseconds的时间内，连续向Sentinel返回无效回复，那么Sentinel会认为该实例主观下线。

> 对于监视同一个主服务器的多个Sentinel来说，down-after-milliseconds选项的值可能不同，因此，当一个Sentinel将主服务器判断为主观下线时，其他Sentinel可能仍会认为主服务器处于在线状态。

## 检查客观下线状态

当Sentinel将一个主服务器判断为主观下线后，会向同样监视这一主服务器的其他Sentinel进行询问该主服务器状态（主观或客观下线）。如果Sentinel接收到足够数量的下线判断后，会将主服务器判断为客观下线，并对主服务器进行故障转移操作。

`SENTINEL is-master-down-by-addr`

**客观下线判断条件**

当认为主服务器已经进入（主观或客观）下线状态的Sentinel的数量，超过Sentinel配置中设置的quorum参数的值，那么该Sentinel就会认为主服务器已经进入客观下线状态。

## 选举Leader Sentinel

当一个主服务器被判断为客观下线时，监视这个下线主服务器的各个Sentinel会进行协商，选举出一个Leader Sentinel，并由Leader Sentinel对下线主服务器执行故障转移操作。（Raft算法的实现）

#### 选举的规则和方法

+ 所有在线的、监视同一个主服务器的Sentinel都有被选为Leader Sentinel的资格。
+ 每次进行Leader Sentinel选举后，不论选举是否成功，所有Sentinel的配置纪元（configuration epoch）都会自增一次。
+ 在一个配置纪元里，所有Sentinel都有一次将某个Sentinel设置为局部Leader Sentinel的机会，并且设置后在该纪元内不变。
+ 每个发现主服务器进入客观下线的Sentinel都会要求其他Sentinel将自己设为局部Leader Sentinel。
+ 当每一个源Sentinel向目标Sentinel发送`is-master-down-by-addr`命令，并且命令中的run_id参数不是*符号而是源Sentinel的run_id时，表示源Sentinel要求目标Sentinel将前者设置为后者的局部Leader Sentinel。
+ Sentinel设置局部Leader Sentinel的规则是先到先得，设置后拒绝后来的请求。
+ 目标Sentinel在接收到SENTINEL is-master-down-by-addr命令后，将向源Sentinel返回一条命令回复，回复中的leader_runid参数和leader_epoch参数分别记录了目标Sentinel的局部Leader Sentinel的run_id和配置纪元。
+ 如果有某个Sentinel被半数以上的Sentinel设置成了局部Leader Sentinel，那么这个Sentinel将成为Leader Sentinel。
+ 如果在给定的时限内，没有Leader Sentinel被选出，则在一段时间后重启选举。

## 故障转移

故障转移包括三个步骤：

1. 在已下线主服务器属下的所有从服务器里，选出一个从服务器，并将其转换为主服务器。
2. 让已下线的主服务器属下的所有从服务器改为复制新的主服务器。
3. 将已下线的主服务器设置为新的主服务器的从服务器，当这个旧的主服务器重新上线时，它就会成为新的主服务器的从服务器。

#### 选出新的主服务器

排除：

1. 下线或短线状态的从服务器
2. 最近5秒内没有回复过Leader Sentinel的
3. 与已下线主服务器连接断开超过down-after-milliseconds * 10毫秒的

然后在剩余的列表中选：

优先级高 -> 复制偏移量最大 -> 运行ID最小

#### 修改从服务器的复制目标

向其他服务器发送SLAVE OF命令，复制新的服务器。

#### 将旧的主服务器变为从服务器

将已下线的主服务器设置为新的从服务器。