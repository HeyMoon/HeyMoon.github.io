---
title: Redis主从-Sentinel配置
date: 2021-12-07 22:22:43
tags:
  - Redis,Redis Master-Slave,Redis Sentinel
categories:
  - tech
---

1. 下载Redis
````
wget http://download.redis.io/releases/redis-6.2.6.tar.gz
````

2. 解压
````
tar zxvf redis-6.2.6.tar.gz
````

3. 进入Redis源码
````
cd redis-6.2.6
````
<!--more-->

4. 编译
````
make
````

编译好之后会在`src`目录生成一些可执行文件.为了方便，我们把一些文件拷贝到`redis`目录。

在 `/usr/local`下创建需要的目录。

````
cd /usr/local
sudo mkdir redis
````

`/usr/local/redis` 目录创建成功，需要修改`redis目录所有者`

````
sudo chown -R user:user redis/
````

然后在`redis`目录下，新建`bin`和`conf`目录。

把前面编译后的redis的可执行文件（在`redis-6.2.6/src`下），复制到`/usr/local/redis/bin`里面去。

````
sudo cp redis* /usr/local/redis/bin/
````

### Master-Slave 搭建
redis给我们提供了一个默认的配置文件`redis.conf`。把它复制到`/usr/local/redis/conf`目录下，并改名为`6379.conf`.

````
cp redis.conf  /usr/local/redis/conf/6379.conf
````

修改配置文件

````
vi 6379.conf
````

修改如下几个配置：

````
port 6379

daemonize no 修改为 daemonize yes (以后台程序方式运行)

pidfile /var/run/redis_6379.pid  改为 pidfile /usr/local/redis/redis_6379.pid(把pidfile生成到有权限的目录下)

logfile /usr/local/redis/redis_6379.log
````

启动：

````
/usr/local/redis/bin/redis-server /usr/local/redis/conf/6379.conf
````

配置从节点：

````
cp 6379.conf 6380.conf
cp 6379.conf 6381.conf
````

修改 `6380.conf`，修改对应端口和pid配置，然后加入`replicaof 127.0.0.1 6379`

````
port 6379 改为 port 6380
pidfile /usr/local/redis/redis_6379.pid   改为 pidfile /usr/local/redis/redis_6380.conf
logfile /usr/local/redis/redis_6380.log
replicaof 127.0.0.1 6379
````

修改 `6381.conf`，修改对应端口和pid配置，然后加入`replicaof 127.0.0.1 6379`

````
port 6379 改为 port 6381
pidfile /usr/local/redis/redis_6379.pid   改为 pidfile /usr/local/redis/redis_6381.conf
logfile /usr/local/redis/redis_6381.log
replicaof 127.0.0.1 6379
````

启动连个redis实例：

````markdown
/usr/local/redis/bin/redis-server  /usr/local/redis/conf/6379.conf
/usr/local/redis/bin/redis-server  /usr/local/redis/conf/6380.conf
/usr/local/redis/bin/redis-server  /usr/local/redis/conf/6381.conf
````

使用 Redis 客户端连接到 6379 端口的 Redis 服务端，使用 `info replication` 查看主从状态

````
/usr/local/redis/bin/redis-cli -c -p 6379

127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:2
slave0:ip=127.0.0.1,port=6380,state=online,offset=154,lag=0
slave1:ip=127.0.0.1,port=6381,state=online,offset=154,lag=0
master_failover_state:no-failover
master_replid:91e2c5ec2ab71d0f8531e8176cf5d197a9420544
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:154
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:154
````

可以看到 6379 的 role 是 master,connected slave 有两个。

设置一个 key 试试：

````
127.0.0.1:6379> set test 111
OK
````

使用 Redis 客户端连接到 6380 端口的 Redis 服务端，使用 `info replication` 查看主从状态

````
/usr/local/redis/bin/redis-cli -c -p 6380

127.0.0.1:6380> info replication
# Replication
role:slave
master_host:127.0.0.1
master_port:6379
master_link_status:up
master_last_io_seconds_ago:1
master_sync_in_progress:0
slave_read_repl_offset:631350
slave_repl_offset:631350
slave_priority:100
slave_read_only:1
replica_announced:1
connected_slaves:0
master_failover_state:no-failover
master_replid:91e2c5ec2ab71d0f8531e8176cf5d197a9420544
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:631350
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:631350
````

查看主服务器设置的 key:

````
127.0.0.1:6380> get test
"111"
````

可以看到从服务器已经同步可主服务器的数据库状态。

### Sentinel 搭建
在 conf 目录下新建一个 `sentinel` 文件夹，并进入：

`mkdir sentinel && cd sentinel`

Sentinel 配置如下所示：

````
port 26379
# 是否后台运行
daemonize yes
pidfile /usr/local/redis/conf/sentinel/redis-sentinel-26379.pid
logfile /usr/local/redis/conf/sentinel/redis-sentinel-26379.log
dir "./"
sentinel monitor mymaster 127.0.0.1 6379 2
acllog-max-len 128
````

新建文件 `sentinel-26379.conf`,将上述配置 copy 到 `sentinel-26379.conf`。

再增加两个 sentinel 节点配置：

````
cp sentinel-26379.conf sentinel-26380.conf
cp sentinel-26379.conf sentinel-26381.conf
````

修改 `sentinel-26380.conf`:

````
port 26380
pidfile /usr/local/redis/conf/sentinel/redis-sentinel-26380.pid
logfile /usr/local/redis/conf/sentinel/redis-sentinel-26380.log
````

修改 `sentinel-26381.conf`:

````
port 26381
pidfile /usr/local/redis/conf/sentinel/redis-sentinel-26381.pid
logfile /usr/local/redis/conf/sentinel/redis-sentinel-26381.log
````

启动 Sentinel, 在 `bin` 目录执行如下命令:

````
➜  bin ./redis-sentinel ../conf/sentinel/sentinel-26379.conf
➜  bin ./redis-sentinel ../conf/sentinel/sentinel-26380.conf
➜  bin ./redis-sentinel ../conf/sentinel/sentinel-26381.conf
````

使用 `ps -ef | grep redis | grep -v grep` 查看 redis 进程

````
  ps -ef | grep redis | grep -v grep
  501 69696     1   0 10:27PM ??         0:07.13 ./redis-server 127.0.0.1:6379
  501 69733     1   0 10:28PM ??         0:06.90 ./redis-server 127.0.0.1:6380
  501 69745     1   0 10:28PM ??         0:06.83 ./redis-server 127.0.0.1:6381
  501 69946     1   0 10:32PM ??         0:08.37 ./redis-sentinel *:26379 [sentinel]
  501 69956     1   0 10:32PM ??         0:08.34 ./redis-sentinel *:26380 [sentinel]
  501 69962     1   0 10:32PM ??         0:08.30 ./redis-sentinel *:26381 [sentinel]
````

测试 Sentinel,我们将端口为 6379 的 Redis kill 掉， `kill -9 69696`,这时我们查看 6380 和 6381 两台 Redis 服务器的状态：

````
➜  bin ./redis-cli -c -p 6380

127.0.0.1:6380> info replication
# Replication
role:slave
master_host:127.0.0.1
master_port:6379
master_link_status:down
master_last_io_seconds_ago:-1
master_sync_in_progress:0
slave_read_repl_offset:864009
slave_repl_offset:864009
master_link_down_since_seconds:18
slave_priority:100
slave_read_only:1
replica_announced:1
connected_slaves:0
master_failover_state:no-failover
master_replid:91e2c5ec2ab71d0f8531e8176cf5d197a9420544
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:864009
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:864009
````

我们可以看到 `master_link_status:down`。过一分钟左右，我们再次查看 6380 的状态：

````
➜  bin ./redis-cli -c -p 6380

127.0.0.1:6380> info replication
# Replication
role:master
connected_slaves:1
slave0:ip=127.0.0.1,port=6381,state=online,offset=871802,lag=0
master_failover_state:no-failover
master_replid:e280894e989aab690fbb112da0735016c992bbed
master_replid2:91e2c5ec2ab71d0f8531e8176cf5d197a9420544
master_repl_offset:871935
second_repl_offset:864010
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:871935
````

我们可以看到 6380 已经成为了新的 master。这是我们再将 6379 启动，并使用 Redis 客户端连接到 6379,可以看到 6379 已经成为了 6380 的 salve。

````
➜  bin ./redis-server ../conf/redis-6379.conf

➜  bin ./redis-cli -c -p 6379
127.0.0.1:6379> info replication
# Replication
role:slave
master_host:127.0.0.1
master_port:6380
master_link_status:up
master_last_io_seconds_ago:1
master_sync_in_progress:0
slave_read_repl_offset:942938
slave_repl_offset:942938
slave_priority:100
slave_read_only:1
replica_announced:1
connected_slaves:0
master_failover_state:no-failover
master_replid:e280894e989aab690fbb112da0735016c992bbed
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:942938
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:941586
repl_backlog_histlen:1353

➜  bin ./redis-cli -c -p 6380
127.0.0.1:6380> info replication
# Replication
role:master
connected_slaves:2
slave0:ip=127.0.0.1,port=6381,state=online,offset=961152,lag=0
slave1:ip=127.0.0.1,port=6379,state=online,offset=961152,lag=1
master_failover_state:no-failover
master_replid:e280894e989aab690fbb112da0735016c992bbed
master_replid2:91e2c5ec2ab71d0f8531e8176cf5d197a9420544
master_repl_offset:961152
second_repl_offset:864010
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:961152
````
