# Redis 复习总攻略



#####主从复制原理：

1：从服务器首先向主服务器的发送sync 

2: 主服务器接收到命令后会生成快照，发送给从服务器slave，同时还可以接收客户端的命令，并将新接收的命令存储到内存中。

3:从服务器接收到数据之后，首先进行持久化，删除原来的快照信息，并将数据存储到内存中

4:主服务器将内存中的数据发送给从服务器

##### （二）数据部分复制

当主从服务器断开连接之后，在redis 2.8 之后支持部分数据复制，从服务器发送psync指令，去除与主服务器master 的相同数据，只进行部分数据的复制（断点续传）

主服务器在内存中创建一个复制数据的缓存队列，主服务器和从服务器都会维护这个队列的下标offset 与matser 进程id。 因此在主从断开连接之后，slave 会进行未完成的复制，当从节点为胡的节点太旧或者mater 进程发生了变化，从姐弟啊及那个进行全量复制。

##### 问题

当一个主节点有大量从节点的时候会出现**主从风暴**，多个从节点主节点同步的时候压力过大，通过从节点下面配置从节点解决

#####redis Lua 脚本

一次性发送多个脚本，将多个指令打包为一个指令一次性发送，减少网络开销，实际上发送多个指令只需要一次网络开销，在进行Lua 脚本的时候 是需要把脚本的指令先缓存起来。

Redis Lua 脚本，在执行命令的时候会将

##### Redis  哨兵模式

创建哨兵，搭建主从，从服务器会先从哨兵中获取主服务器的信息，但是在后面直接会直接从主服务器上获取信息，但是当主服务器信息发生变化的时候，哨兵是首先感知到的，当主服务器挂掉之后首先哨兵服务器感知到，然后进行重新选举。

##### Redis 集群原理

redis 集群是分为16384 个节点，分布在不同的节点上，当客户端与集群创建链接之后，客户端获取到集群的槽位信息并缓存到本地形成映射，当客户端获取数据的时候，直接到指定目标的节点获取数据。

问题：可能会出现槽位信息与客户端存储的映射信息不一致，需要调整。还有一种情况，当客户端请求的数据的槽位不是存储的槽位节点的时候，服务器集群会将正确的槽位信息反馈给客户端，客户端到指定的槽位节点获取数据，并且更新槽位信息。

##### Redis集群节点的通讯机制

######采用的是集中式或者gossip 协议

集中式：集中式优点在于当服务器节点发生变化的时候，所有节点会马上感知到，不足在于，所有的节点会存储在一个地方会造成这个地方压力变大。

gossip 协议包含多种消息（meet,fail ,ping,pong）

meet:某个节点向新加入的节点发送meet,让心姐弟啊加入到集群中，新节点可以与其他节点通讯

ping: 每个节点频繁地给其他节点送ping,包括自己的状态和自己的元数据，相互之间通过ping 保持通讯交换元数据（包括 hash slot 信息，集群节点的增加或者移除）

pong:对ping  和 meet  做出的反应

fail:当某个节点得知指定节点宕机之后会通知所有节点

特点：相比与集中通讯协议，gossip 协议通讯元数据的更新会比较分散，更新会陆陆续续打到所有节点上有一定的延时，降低了压力 

#####通讯端口 170001 在端口号前加10000

##### 网络抖动

网络抖动比较常见，为了避免由于网络抖动导致redis 集群不可链接但是又很快恢复正常，可以使用redis_node_timeout ,当超过设置的时间才会认定该节点出现故障。

###### redis集群选举机制

1.当主节点宕机之后，他的slave 尝试及逆行failover (当slave 发现他的主节点处于fail状态的时候)，可能一个master会存在多个slave,多个salve 进行竞争。

2. slave 将自己集群的currentEpoch +1 ，并产生FAILOVER_AUTH_REQUEST
3. 只有master 对其进行响应，并对每一个epoch 制作一次响应，并返回FAILOVER_AUTH_ACK
4. 当slave超过半数的ack  之后选举为master 
5. slave 通过pong 广播给其他节点

###### 缓存穿透

缓存穿透是指查询一个本来就不存在的数据，缓存层与和存储层都不会命中，如果从存储层拆卸拿不到数据就不会写入。缓存穿透是将每一次都不存在的数据都会到存储层查询。

###### 原因

数据出现问题，或有人恶意爬虫

###### 解决办法

1、缓存空对象

2、布隆过滤器：布隆过滤器当说某个值存在的时候这个值不一定存在，但是说这个值不存在的时候那么这个值是一定不存在的

###### 缓存失效 / 缓存击穿

当有大量的缓存key 同时失效，可能导致大量请求同时访问数据库，可能导致数据库压力瞬间增大，对于这总情况需要在设置缓存失效的时候设置为不同的时间

###### 缓存雪崩

由于某些原因缓存层支撑不住或者宕掉，缓存层不起作用，当大量请求过来直接进入存储层，存储层的访问量会出现暴增，导致存储层出现级联宕机情况

###### 解决缓存雪崩方案

方案一：使用高可用的缓存，使用redis 集群模式

方案二：使用隔离组件使用限流，sentinel 与hystrix进行限流

###### 热点数据 缓存策略

一般情况下使用  key + 过期时间，但是当有大量线程同时访问的时候，当重建缓存的时候计算比较复杂，造成后端服务压力变大。为了避免大量线程重新构建缓存，只允许一个线程进行构建

###### 数据库双写不一致情况

1、数据库与缓存不一致，在一般情况下不会出现但是在大量并发的时候会出现，可以给缓存数据加上过期时间。

2、家伙是那个过期时间能够解决一部分问题，但是如果不允许出现缓存不一致的问题，可以加上读写锁，保证读写或者写写的时候排好队，读读的时候相当于没有锁

3、可以使用阿里开源canel 中间件进行监听，但是会增加系统复杂度

 







