---
title: Redis Cluster
date: 2021-12-07 22:23:10
tags:
categories:
---

当公司业务发展，数据量增加到一定程度后，我们总是绕不开分布式这个话题。这个问题牵扯很多方面，

+ 分区策略（Sharding）

+ 数据备份：数据备份什么时候做？粒度是什么？怎样备份？

+ 分区再平衡：当数据分布发生拓扑变化的时候，怎么把数据从原来的节点迁移到新的节点上？

+ 集群管理：如何管理整个集群，如何把用户请求定向到某个特定的节点上？

这些问题很很多不同的解法，比如分片策略，不同的数据库可能选择不同的分片策略，比如 Hbase 采用基于关键字区间的分区，Cassandra,MongoDB,Redis 采用基于关键字哈希值分区。我们不妨来看看开源社区中使用最普遍的分布式解决方案之一：Redis Cluster，看看它是如何解决分布式的问题。

## Redis Cluster
Redis Cluster是一个Redis的分布式部署形式，使用数据分区的办法把数据分配到不同的节点；每个节点可以有自己的备份节点（一个或多个）。整个集群之上另有一个叫做Redis Sentinel的分布式组件用以提供更丰富的HA能力。

### 数据分区
Redis Cluster 使用 Slot(槽)的概念，集群的整个数据库被分为 16384 个 slot，16384个 slot 可以分给集群中的多个节点。Redis 把每个key的值 hash(CRC16) 成 0 ~ 16383 之间的一个数。这个 hash 值被用来确定对应的数据存储在哪个 slot 。集群中的每个节点都存储了一份类似路由表的东西，描述每个节点所拥有的 Slots。假设集群里有3个节点(node1, node2, node3)，我们可以这样分配 slot ：

````
node1: 0 ~ 5000
node2: 5001 ~ 10000
node3: 10001 ~ 16383
````

Redis 使用一下算法来计算给定的 key 属于哪个槽：

````
def slot_number(key):
    return CRC16(key) & 16383
````

Slot 是集群数据管理和迁移的最小单位，保证数据管理的粒度易于管理。集群中的每个节点都知道 Slot 在集群中的分布，节点之间使用 Gossip 协议通信。

当数据库中的 16384 个槽都有节点在处理时，集群处于上线状态(ok)；如果数据库中有任何一个槽没有得到处理，那么集群处于下线状态(fail)。

### 数据备份
Redis的备份是最简单的 Master-Slave 异步复制(AOF)备份。每个 Master 节点都可以有若干个 Slave 跟随；Slave（Replica）可以提供高可靠性（HA)，也可以用作只读节点提供高吞吐量。

### 分区再平衡
分区再平衡，也叫重新分片(Reshard)一般是因为有节点的变化（有节点下线，或者有新的节点加入）。简单的讲，Reshard就是把一些slots从一个节点转移到另一个节点。

Reshard 的原理并不复杂，基于 redis-trib 对集群的单个 slot 进行重新分片的步骤如下：

+ redis-trib 对目标节点发送 Cluster setslot <slot> importing <source_id> 命令，让目标节点准备好从源节点导入(import) 属于槽 slot 的键值对

+ redis-trib 对源节点发送 Cluster setslot <slot> migrating <target_id> 命令，让源节点准备好将属于槽 slot 的键值对迁移(migrate)至目标节点

+ redis-trib 将键值对从源节点迁移至目标节点

+ redis-trib 将槽 slot 指派给集群中的任意一个节点，并通过消息发送至整个集群，最终集群中所有节点都会知道槽 slot 被指派给了目标节点。

### 集群管理
Redis Cluster 集群管理引入了一个新的组件，叫做 Redis Sentinel，在整个集群的纬度上提供高可用的能力。简单的讲，它类似一个集群的Registry，包含监控、报警、自动切换、配置管理等常见功能。另外，Sentinel本身也是分布式部署，采用 Raft 算法维持状态的一致性。

+ Sentinels 监视所有的数据节点
+ Sentinels 监视所有其他 Sentinels
+ 当 Sentinels 对节点宕机达成共识之后，选举出一个新的master（升级）并完成故障转移

### Redis Cluster 配置

````
bind 127.0.0.1 -::1
port 7379
daemonize yes
pidfile /Users/dengyunhui/Documents/install/redis-6.2.6/config/cluster/redis_7379.pid
loglevel debug
logfile /Users/dengyunhui/Documents/install/redis-6.2.6/config/cluster/redis_7379.log
cluster-enabled yes
cluster-config-file nodes-7379.conf
cluster-node-timeout 15000
````

新建文件 `cluster-7379.conf`,同时复制出五个节点配置：

````
cp cluster-7379.conf cluster-7380.conf
cp cluster-7379.conf cluster-7381.conf
cp cluster-7379.conf cluster-7382.conf
cp cluster-7379.conf cluster-7383.conf
cp cluster-7379.conf cluster-7384.conf
````

删除数据目录下的所有文件,启动 6 个节点：

````
➜  bin ./redis-server ../config/cluster/cluster-7379.conf
➜  bin ./redis-server ../config/cluster/cluster-7380.conf
➜  bin ./redis-server ../config/cluster/cluster-7381.conf
➜  bin ./redis-server ../config/cluster/cluster-7382.conf
➜  bin ./redis-server ../config/cluster/cluster-7383.conf
➜  bin ./redis-server ../config/cluster/cluster-7384.conf
````

如果在之前的集群中的某个节点存储过数据或者集群配置没有删除，会报类似错误 ** [ERR] Node 127.0.0.1:7379 is not empty. Either the node already knows other nodes (check with CLUSTER NODES) or contains some key in database 0. ** 如果集群配置未删除，则删除所有集群配置，否则进入之前存储数据的客户端执行 flushall 和 cluster reset 清空数据并重置集群。
然后重新启动所有节点。只需键入以下内容即可为 Redis 6 创建集群：

````
../../bin/redis-cli --cluster create 127.0.0.1:7379 127.0.0.1:7380 127.0.0.1:7381 127.0.0.1:7382 127.0.0.1:7383 127.0.0.1:7384 --cluster-replicas 1
````

create 后面跟着的是6个节点的地址和端口，选项--cluster-replicas 1意味着为每个主节点都提供一个从节点。出现如下结果即表示创建成功：

````
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 127.0.0.1:7383 to 127.0.0.1:7379
Adding replica 127.0.0.1:7384 to 127.0.0.1:7380
Adding replica 127.0.0.1:7382 to 127.0.0.1:7381
>>> Trying to optimize slaves allocation for anti-affinity
[WARNING] Some slaves are in the same host as their master
M: 4ebf3cedca6bcd9ebcadcdaf8d6f591656cf9b4a 127.0.0.1:7379
   slots:[0-5460] (5461 slots) master
M: 6c9e8b7d86e466d0d63c9ceb6603441126158c0b 127.0.0.1:7380
   slots:[5461-10922] (5462 slots) master
M: 82b9f8c549d79c99265dbe89ed286413a260826a 127.0.0.1:7381
   slots:[10923-16383] (5461 slots) master
S: c79437759320e52f5838b1c942ef9ba90b20e32f 127.0.0.1:7382
   replicates 4ebf3cedca6bcd9ebcadcdaf8d6f591656cf9b4a
S: bfc699dbe7967ea0d41dcef430b926ae988ce91a 127.0.0.1:7383
   replicates 6c9e8b7d86e466d0d63c9ceb6603441126158c0b
S: 7919d0bc8c0cd1fc2966ca770106671a97787f1a 127.0.0.1:7384
   replicates 82b9f8c549d79c99265dbe89ed286413a260826a
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
.
>>> Performing Cluster Check (using node 127.0.0.1:7379)
M: 4ebf3cedca6bcd9ebcadcdaf8d6f591656cf9b4a 127.0.0.1:7379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: c79437759320e52f5838b1c942ef9ba90b20e32f 127.0.0.1:7382
   slots: (0 slots) slave
   replicates 4ebf3cedca6bcd9ebcadcdaf8d6f591656cf9b4a
M: 6c9e8b7d86e466d0d63c9ceb6603441126158c0b 127.0.0.1:7380
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
M: 82b9f8c549d79c99265dbe89ed286413a260826a 127.0.0.1:7381
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: bfc699dbe7967ea0d41dcef430b926ae988ce91a 127.0.0.1:7383
   slots: (0 slots) slave
   replicates 6c9e8b7d86e466d0d63c9ceb6603441126158c0b
S: 7919d0bc8c0cd1fc2966ca770106671a97787f1a 127.0.0.1:7384
   slots: (0 slots) slave
   replicates 82b9f8c549d79c99265dbe89ed286413a260826a
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
````

使用 Redis 客户端登录 7379 服务器,使用 `cluster info` 查看集群状态：

````
➜  bin ./redis-cli -c -p 7379
127.0.0.1:7379> cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:1
cluster_stats_messages_ping_sent:2266
cluster_stats_messages_pong_sent:2152
cluster_stats_messages_sent:4418
cluster_stats_messages_ping_received:2147
cluster_stats_messages_pong_received:2266
cluster_stats_messages_meet_received:5
cluster_stats_messages_received:4418
````

### Redis cluster 使用

#### redis-cli 以集群方式启动
redis-cli 记得要加 `-c` 以集群模式启动。

````
➜  bin ./redis-cli -c -p 7379

127.0.0.1:7379> set fruits apple
-> Redirected to slot [14943] located at 127.0.0.1:7381
OK
````

当在 7379 使用 `set` 来设置 `fruits` 的值时，对应的槽为 14943，这个槽是属于 7381 节点负责的，所以集群模式会自动重定向到 7381 节点来完成 set key 命令。

#### 重新分片
重新分片其实也就是槽的迁移。将 Slot 从一个或几个节点迁移到另一个节点。
Redis官方也是提供了命令用来重新分片的，如果 Redis 的版本是 3 或者 4 的话使用redis-trib.rb这个工具，如果 Redis 的版本为 5-6的话，可以使用如下命令：

````
redis-cli --cluster reshard 127.0.0.1:7379
````

在这里只需要指定一个节点，redis-cli将自动找到其他节点。
首先会测试集群的运行状态，然后询问你要重新分配多少个槽(我这里指定了1000个槽)。 <span style="background-color: #FF0000">How many slots do you want to move (from 1 to 16384)?</span> 然后我们需要指定重新分片的目标ID(也就是指定哪个节点来接收这些重新分配的槽)。可以通过以下命令来查看某个节点的信息(里面包括了节点的ID)。

````
redis-cli -p 7379 cluster nodes | grep myself
872e3876ea53a1ba1a77af3fb5a8c72704ac0eab 127.0.0.1:7379@16379 myself,master - 0 1580801413000 7 connected 0-5961 10923-11421
````

最终需要指定从哪些节点获取槽(这里简单的指定all也就是所有的节点)。就会开始执行重新分片了，通过终端可以看到槽正在从一个节点移动到另一个节点。
最后检查集群状态可以发现目标节点的槽数量变成了6461

输入 `cluster slots` 查看集群节点 slot 的分配情况：

````
127.0.0.1:7379> cluster slots
1) 1) (integer) 0
   2) (integer) 5460
   3) 1) "127.0.0.1"
      2) (integer) 7379
      3) "4ebf3cedca6bcd9ebcadcdaf8d6f591656cf9b4a"
   4) 1) "127.0.0.1"
      2) (integer) 7382
      3) "c79437759320e52f5838b1c942ef9ba90b20e32f"
2) 1) (integer) 5461
   2) (integer) 10922
   3) 1) "127.0.0.1"
      2) (integer) 7380
      3) "6c9e8b7d86e466d0d63c9ceb6603441126158c0b"
   4) 1) "127.0.0.1"
      2) (integer) 7383
      3) "bfc699dbe7967ea0d41dcef430b926ae988ce91a"
3) 1) (integer) 10923
   2) (integer) 16383
   3) 1) "127.0.0.1"
      2) (integer) 7381
      3) "82b9f8c549d79c99265dbe89ed286413a260826a"
   4) 1) "127.0.0.1"
      2) (integer) 7384
      3) "7919d0bc8c0cd1fc2966ca770106671a97787f1a"
````

#### 故障转移
先查看当前集群中的节点：

````
➜  cluster ../../bin/redis-cli -p 7379 cluster nodes
bfc699dbe7967ea0d41dcef430b926ae988ce91a 127.0.0.1:7383@17383 slave 6c9e8b7d86e466d0d63c9ceb6603441126158c0b 0 1638885135833 2 connected
82b9f8c549d79c99265dbe89ed286413a260826a 127.0.0.1:7381@17381 master - 0 1638885134819 3 connected 10923-16383
6c9e8b7d86e466d0d63c9ceb6603441126158c0b 127.0.0.1:7380@17380 master - 0 1638885136847 2 connected 5461-10922
7919d0bc8c0cd1fc2966ca770106671a97787f1a 127.0.0.1:7384@17384 slave 82b9f8c549d79c99265dbe89ed286413a260826a 0 1638885133810 3 connected
c79437759320e52f5838b1c942ef9ba90b20e32f 127.0.0.1:7382@17382 slave 4ebf3cedca6bcd9ebcadcdaf8d6f591656cf9b4a 0 1638885133000 1 connected
4ebf3cedca6bcd9ebcadcdaf8d6f591656cf9b4a 127.0.0.1:7379@17379 myself,master - 0 1638885134000 1 connected 0-5460 [14943-<-82b9f8c549d79c99265dbe89ed286413a260826a]
````

从上面可以看出集群中节点：

````
7379(master) -> 7382(slave)
7380(master) -> 7383(slave)
7381(master) -> 7384(slave)
````

我们使用命令 `debug segfault` 让 7380 节点崩溃，我们预计 7380 崩溃之后，7383 会成为新的 master 节点。

````
./redis-cli -p 7380 debug segfault
````

然后等待一段时间，再次使用 `cluster nodes` 查看集群节点，我们发现已经自动完成了故障转移：

````
➜  bin ./redis-cli -p 7379 cluster nodes
bfc699dbe7967ea0d41dcef430b926ae988ce91a 127.0.0.1:7383@17383 master - 0 1638885610213 7 connected 5461-10922
82b9f8c549d79c99265dbe89ed286413a260826a 127.0.0.1:7381@17381 master - 0 1638885608000 3 connected 10923-16383
6c9e8b7d86e466d0d63c9ceb6603441126158c0b 127.0.0.1:7380@17380 master,fail - 1638885553442 1638885548000 2 disconnected
7919d0bc8c0cd1fc2966ca770106671a97787f1a 127.0.0.1:7384@17384 slave 82b9f8c549d79c99265dbe89ed286413a260826a 0 1638885609197 3 connected
c79437759320e52f5838b1c942ef9ba90b20e32f 127.0.0.1:7382@17382 slave 4ebf3cedca6bcd9ebcadcdaf8d6f591656cf9b4a 0 1638885608182 1 connected
4ebf3cedca6bcd9ebcadcdaf8d6f591656cf9b4a 127.0.0.1:7379@17379 myself,master - 0 1638885608000 1 connected 0-5460 [14943-<-82b9f8c549d79c99265dbe89ed286413a260826a]
````

再次启动 7380 ，cluster 会重新将它加入集群，并让它成为从节点最少的那个master节点的从节点(在这里就是 7383)。

````
➜  bin ./redis-cli -p 7379 cluster nodes
bfc699dbe7967ea0d41dcef430b926ae988ce91a 127.0.0.1:7383@17383 master - 0 1638885777395 7 connected 5461-10922
82b9f8c549d79c99265dbe89ed286413a260826a 127.0.0.1:7381@17381 master - 0 1638885777000 3 connected 10923-16383
6c9e8b7d86e466d0d63c9ceb6603441126158c0b 127.0.0.1:7380@17380 slave bfc699dbe7967ea0d41dcef430b926ae988ce91a 0 1638885776384 7 connected
7919d0bc8c0cd1fc2966ca770106671a97787f1a 127.0.0.1:7384@17384 slave 82b9f8c549d79c99265dbe89ed286413a260826a 0 1638885776000 3 connected
c79437759320e52f5838b1c942ef9ba90b20e32f 127.0.0.1:7382@17382 slave 4ebf3cedca6bcd9ebcadcdaf8d6f591656cf9b4a 0 1638885778406 1 connected
4ebf3cedca6bcd9ebcadcdaf8d6f591656cf9b4a 127.0.0.1:7379@17379 myself,master - 0 1638885776000 1 connected 0-5460 [14943-<-82b9f8c549d79c99265dbe89ed286413a260826a]
````

#### 手动故障转移
随便进入一个从节点(只能是从节点)中，执行 `cluster failover` 进行手动故障转移(手动故障转移只是将主节点和从节点的关系换了一下)。

#### 增加/删除节点
##### 增加节点
按照之前的配置，新增加一个节点 cluster-7385.conf ，然后启动它。使用以下命令将它加入到集群中。

````
# 127.0.0.1:7385 指的是新加入集群中的节点的ip和端口
# 127.0.0.1:7379 指的是集群中的某个节点的ip和地址(随便哪个节点都行)
redis-cli --cluster add-node 127.0.0.1:7385 127.0.0.1:7379
````

其实这个命令的作用和cluster meet是一样的，就是用于节点握手通信(唯一不一样的就是执行前会检查集群的状态)。
还记得之前手动搭建集群的时候，节点握手完成后需要给节点分配槽。因为这里的16384个槽都已经分完了，所以这里使用重新分片来给新节点分配槽(新节点就是重新分片的目标节点)。

#### 删除节点
要删除一个从节点，只需使用del-node命令：

````
# 127.0.0.1:7379 是集群中的任意一个节点Ip和端口
# node-id就是要删除的节点ID
redis-cli --cluster del-node 127.0.0.1:7379 <node-id>
````

也可以用相同的方法删除主节点，但是要删除主节点，它必须为空。如果主节点不为空，则需要先将数据从其重新分片到所有其他主节点。否则可以使用手动故障转移的方式将主节点降为从节点，然后再执行删除。

这里以删除之前添加的 7385 节点为例。该节点共有1999个槽，所以在我们删除节点之前需要将这些槽迁移到其它节点上。这里我将槽全部迁移到目标节点 7384 上，将被删除的节点 7385 作为源节点。

redis-cli --cluster reshard 127.0.0.1:6374

````
How many slots do you want to move (from 1 to 16384)? 1999
What is the receiving node ID?
f96b0261c775cf10a254f28374a4ed73c2977b1b # 目标节点redis-6375的ID
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1: 533168c99a03b5593408818dabc1db36396a221f # 源节点redis-6373的ID
Source node #2: done # 只有一个源节点，输入done结束
````

复制代码迁移结束后检查集群状态,可以发现redis-6373节点的槽为0，然后就可以放心的删除节点啦。

````
redis-cli --cluster check 127.0.0.1:6375
127.0.0.1:6375 (f96b0261...) -> 0 keys | 6356 slots | 1 slaves.
127.0.0.1:6374 (10e8962e...) -> 1 keys | 5672 slots | 1 slaves.
127.0.0.1:6373 (533168c9...) -> 0 keys | 0 slots | 0 slaves.
127.0.0.1:6376 (e80c00d0...) -> 0 keys | 4356 slots | 1 slaves.
````

复制代码安全的将redis-6373节点从集群中删除（通过日志可以发现，其实redis是先调用 cluster forget忘记节点，然后再将节点下线）

````
redis-cli --cluster del-node 127.0.0.1:6375 533168c99a03b5593408818dabc1db36396a221f
>>> Removing node 533168c99a03b5593408818dabc1db36396a221f from cluster 127.0.0.1:6375
>>> Sending CLUSTER FORGET messages to the cluster...
>>> SHUTDOWN the node.
````


参考：

[Redis Tutorial](https://redis.io/topics/cluster-tutorial)
