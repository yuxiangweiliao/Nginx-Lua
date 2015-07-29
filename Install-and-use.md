# Redis/SSDB+Twemproxy 安装与使用  
  
目前对于互联网公司不使用 Redis 的很少，Redis 不仅仅可以作为 key-value 缓存，而且提供了丰富的数据结果如 set、list、map 等，可以实现很多复杂的功能；但是 Redis 本身主要用作内存缓存，不适合做持久化存储，因此目前有如 SSDB、ARDB 等，还有如京东的 JIMDB，它们都支持 Redis 协议，可以支持 Redis 客户端直接访问；而这些持久化存储大多数使用了如LevelDB、RocksDB、LMDB 持久化引擎来实现数据的持久化存储；京东的 JIMDB 主要分为两个版本：LevelDB 和 LMDB，而我们看到的京东商品详情页就是使用 LMDB 引擎作为存储的，可以实现海量KV存储；当然 SSDB 在京东内部也有些部门在使用；另外调研过得如豆瓣的 beansDB 也是很不错的。具体这些持久化引擎之间的区别可以自行查找资料学习。
 
Twemproxy 是一个 Redis/Memcached 代理中间件，可以实现诸如分片逻辑、HashTag、减少连接数等功能；尤其在有大量应用服务器的场景下 Twemproxy 的角色就凸显了，能有效减少连接数。
 
## Redis 安装与使用  
 
### 下载 redis 并安装  

Java **代码**    
  
```
cd /usr/servers/  
wget https://github.com/antirez/redis/archive/2.8.19.tar.gz  
tar -xvf 2.8.19.tar.gz  
cd redis-2.8.19/  
make     
```  

通过如上步骤构建完毕。
 
### 后台启动 Redis 服务器  

Java **代码**    
  
```
nohup /usr/servers/redis-2.8.19/src/redis-server  /usr/servers/redis-2.8.19/redis.conf &  
```  
 
### 查看是否启动成功  

Java **代码**  
   
```
ps -aux | grep redis  
```  

### 进入客户端  

Java **代码**    
  
```
/usr/servers/redis-2.8.19/src/redis-cli  -p 6379  
```  
 
### 执行如下命令    

Java **代码**  
  
```
127.0.0.1:6379> set i 1  
OK  
127.0.0.1:6379> get i  
"1"     
```  

通过如上命令可以看到我们的 Redis 安装成功。更多细节请参考 [http://redis.io/](http://redis.io/)。
 
## SSDB 安装与使用  

### 下载 SSDB 并安装  

Java **代码**   
  
```
\#首先确保安装了g++，如果没有安装，如ubuntu可以使用如下命令安装  
apt-get install g++  
cd /usr/servers  
wget https://github.com/ideawu/ssdb/archive/1.8.0.tar.gz  
tar -xvf 1.8.0.tar.gz  
make   
```  

### 后台启动 SSDB 服务器   

Java **代码**  
  
```
nohup /usr/servers/ssdb-1.8.0/ssdb-server  /usr/servers/ssdb-1.8.0/ssdb.conf &  
```  
 
### 查看是否启动成功  
   
Java **代码**   
  
``` 
ps -aux | grep ssdb  
```  
 
### 进入客户端  

Java **代码**    
  
```
/usr/servers/ssdb-1.8.0/tools/ssdb-cli -p 8888  
/usr/servers/redis-2.8.19/src/redis-cli  -p 888  
```  
   
因为 SSDB 支持 Redis 协议，所以用 Redis 客户端也可以访问 
 
### 执行如下命令  

Java **代码**   
  
```
127.0.0.1:8888> set i 1  
OK  
127.0.0.1:8888> get i  
"1"    
```  

安装过程中遇到错误请参考  [http://ssdb.io/docs/zh_cn/install.html](http://ssdb.io/docs/zh_cn/install.html)；对于 SSDB 的配置请参考官方文档 https://github.com/ideawu/ssdb[http://ssdb.io/docs/zh_cn/install.html](http://ssdb.io/docs/zh_cn/install.html)。
 
## Twemproxy 安装与使用  

首先需要安装 autoconf、automake、libtool 工具，比如 ubuntu 可以使用如下命令安装  

Java **代码**  
  
```
apt-get install autoconf automake  
apt-get install libtool  
```  
  
### 下载 Twemproxy 并安装  

Java **代码**    
  
```
cd /usr/servers  
wget https://github.com/twitter/twemproxy/archive/v0.4.0.tar.gz  
tar -xvf v0.4.0.tar.gz    
cd twemproxy-0.4.0/  
autoreconf -fvi  
./configure && make    
```  
 
此处根据要注意，如上安装方式在有些服务器上可能在大量如mset时可能导致 Twemproxy 崩溃，需要使用如 CFLAGS="-O1" ./configure && make 或 CFLAGS="-O3 -fno-strict-aliasing" ./configure && make 安装。
 
### 配置  

Java **代码**   
  
``` 
vim /usr/servers/twemproxy-0.4.0/conf/nutcracker.yml  
```  
 
Java **代码**    
  
```
server1:  
  listen: 127.0.0.1:1111  
  hash: fnv1a_64  
  distribution: ketama  
  redis: true  
  servers:  
   - 127.0.0.1:6379:1  
```  

### 启动 Twemproxy 代理  

Java **代码**  
  
```
/usr/servers/twemproxy-0.4.0/src/nutcracker  -d -c /usr/servers/twemproxy-0.4.0/conf/nutcracker.yml    
```  

-d 指定后台启动  -c指定配置文件；此处我们指定了代理端口为 1111，其他配置的含义后续介绍。
 
### 查看是否启动成功  
  
Java **代码**   
  
```
ps -aux | grep nutcracker  
```  
 
### 进入客户端  

Java **代码**  
  
```  
/usr/servers/redis-2.8.19/src/redis-cli  -p 1111  
```  
 
### 执行如下命令   

Java 代码    
  
```
127.0.0.1:1111> set i 1  
OK  
127.0.0.1:1111> get i  
"1"     
```  

Twemproxy 文档请参考 [https://github.com/twitter/twemproxy](https://github.com/twitter/twemproxy)。
 
到此基本的安装就完成了。接下来做一些介绍。
 
## Redis 设置  

### 基本设置  

Java **代码**  
  
```
\#端口设置，默认6379    
port 6379    
\#日志文件，默认/dev/null    
logfile ""     
```  

### Redis 内存  
  
Java **代码**  
  
```
内存大小对应关系   
\# 1k => 1000 bytes  
\# 1kb => 1024 bytes  
\# 1m => 1000000 bytes  
\# 1mb => 1024*1024 bytes  
\# 1g => 1000000000 bytes  
\# 1gb => 1024*1024*1024 bytes  
  
\#设置Redis占用100mb的大小  
maxmemory  100mb  
  
\#如果内存满了就需要按照如相应算法进行删除过期的/最老的  
\#volatile-lru 根据LRU算法移除设置了过期的key  
\#allkeys-lru  根据LRU算法移除任何key(包含那些未设置过期时间的key)  
\#volatile-random/allkeys->random 使用随机算法而不是LRU进行删除  
\#volatile-ttl 根据Time-To-Live移除即将过期的key   
\#noeviction   永不过期，而是报错  
maxmemory-policy volatile-lru  
  
\#Redis并不是真正的LRU/TTL，而是基于采样进行移除的，即如采样10个数据移除其中最老的/即将过期的  
maxmemory-samples 10   
```  
 
而如 Memcached 是真正的 LRU，此处要根据实际情况设置缓存策略，如缓存用户数据时可能带上了过期时间，此时采用 volatile-lru 即可；而假设我们的数据未设置过期时间，此时可以考虑使用 allkeys-lru/allkeys->random；假设我们的数据不允许从内存删除那就使用noeviction。
 
内存大小尽量在系统内存的 60%~80% 之间，因为如客户端、主从时复制时都需要缓存区的，这些也是耗费系统内存的。
 
Redis 本身是单线程的，因此我们可以设置每个实例在 6-8GB 之间，通过启动更多的实例提高吞吐量。如 128GB 的我们可以开启 8GB * 10 个实例，充分利用多核 CPU。
 
### Redis 主从  

实际项目时，为了提高吞吐量，我们使用主从策略，即数据写到主 Redis，读的时候从从 Redis上读，这样可以通过挂载更多的从来提高吞吐量。而且可以通过主从机制，在叶子节点开启持久化方式防止数据丢失。  

Java **代码**   
  
```
\#在配置文件中挂载主从，不推荐这种方式，我们实际应用时Redis可能是会宕机的  
slaveof masterIP masterPort  
\#从是否只读，默认yes  
slave-read-only yes  
\#当从失去与主的连接或者复制正在进行时，从是响应客户端（可能返回过期的数据）还是返回“SYNC with master in progress”错误，默认yes响应客户端  
slave-serve-stale-data yes  
\#从库按照默认10s的周期向主库发送PING测试连通性  
repl-ping-slave-period 10  
\#设置复制超时时间（SYNC期间批量I/O传输、PING的超时时间），确保此值大于repl-ping-slave-period  
\#repl-timeout 60  
\#当从断开与主的连接时的复制缓存区，仅当第一个从断开时创建一个，缓存区越大从断开的时间可以持续越长  
\# repl-backlog-size 1mb  
\#当从与主断开持续多久时清空复制缓存区，此时从就需要全量复制了，如果设置为0将永不清空    
\# repl-backlog-ttl 3600  
\#slave客户端缓存区，如果缓存区超过256mb将直接断开与从的连接，如果持续60秒超过64mb也会断开与从的连接  
client-output-buffer-limit slave 256mb 64mb 60   
此处需要根据实际情况设置client-output-buffer-limit slave和 repl-backlog-size；比如如果网络环境不好，从与主经常断开，而每次设置的数据都特别大而且速度特别快（大量设置html片段）那么就需要加大repl-backlog-size。 
```  
 
主从示例  

Java **代码**   
  
```
cd /usr/servers/redis-2.8.19  
cp redis.conf redis_6660.conf  
cp redis.conf redis_6661.conf  
vim redis_6660.conf  
vim redis_6661.conf   
```  
 
将端口分别改为 port 6660 和 port 6661，然后启动  

Java **代码**  
  
```
nohup /usr/servers/redis-2.8.19/src/redis-server  /usr/servers/redis-2.8.19/redis_6660.conf &  
nohup /usr/servers/redis-2.8.19/src/redis-server  /usr/servers/redis-2.8.19/redis_6661.conf &    
```  

查看是否启动  

Java **代码**  
  
```
ps -aux | grep redis  
```  
 
进入从客户端，挂主  

Java **代码**   
  
```
/usr/servers/redis-2.8.19/src/redis-cli  -p 6661    
```  

Java **代码**  
  
```
127.0.0.1:6661> slaveof 127.0.0.1 6660  
OK  
127.0.0.1:6661> info replication  
\# Replication  
role:slave  
master_host:127.0.0.1  
master_port:6660  
master_link_status:up  
master_last_io_seconds_ago:3  
master_sync_in_progress:0  
slave_repl_offset:57  
slave_priority:100  
slave_read_only:1  
connected_slaves:0  
master_repl_offset:0  
repl_backlog_active:0  
repl_backlog_size:1048576  
repl_backlog_first_byte_offset:0  
repl_backlog_histlen:0   
```   
 
进入主  

Java **代码**     
  
```
/usr/servers/redis-2.8.19# /usr/servers/redis-2.8.19/src/redis-cli  -p 6660  
```  
  
Java **代码**    
  
```
127.0.0.1:6660> info replication  
\# Replication  
role:master  
connected_slaves:1  
slave0:ip=127.0.0.1,port=6661,state=online,offset=85,lag=1  
master_repl_offset:85  
repl_backlog_active:1  
repl_backlog_size:1048576  
repl_backlog_first_byte_offset:2  
repl_backlog_histlen:84  
127.0.0.1:6660> set i 1  
OK   
```  

进入从   
 
Java 代码   
  
```
/usr/servers/redis-2.8.19/src/redis-cli  -p 6661    
```  

Java 代码   
   
```
127.0.0.1:6661> get i  
"1"     
```  

此时可以看到主从挂载成功，可以进行主从复制了。使用 slaveof no one 断开主从。
 
### Redis 持久化  

Redis 虽然不适合做持久化存储，但是为了防止数据丢失有时需要进行持久化存储，此时可以挂载一个从（叶子节点）只进行持久化存储工作，这样假设其他服务器挂了，我们可以通过这个节点进行数据恢复。  

Redis 持久化有 RDB 快照模式和 AOF 追加模式，根据自己需求进行选择。
 
### RDB 持久化  

Java **代码**  
  
```  
\#格式save seconds changes 即N秒变更N次则保存，从如下默认配置可以看到丢失数据的周期很长，通过save “” 配置可以完全禁用此持久化  
save 900 1    
save 300 10    
save 60 10000   
\#RDB是否进行压缩，压缩耗CPU但是可以减少存储大小  
rdbcompression yes  
\#RDB保存的位置，默认当前位置    
dir ./  
\#RDB保存的数据库名称  
dbfilename dump.rdb    
\#不使用AOF模式，即RDB模式  
appendonly no     
```  

可以通过 set 一个数据，然后很快的 kill 掉 redis 进程然后再启动会发现数据丢失了。
 
### AOF 持久化   
   
AOF（append only file）即文件追加模式，即把每一个用户操作的命令保存下来，这样就会存在好多重复的命令导致恢复时间过长，那么可以通过相应的配置定期进行 AOF 重写来减少重复。  

Java 代码   
   
``` 
\#开启AOF  
appendonly yes  
\#AOF保存的位置，默认当前位置    
dir ./  
\#AOF保存的数据库名称  
appendfilename appendonly.aof  
\#持久化策略，默认每秒fsync一次，也可以选择always即每次操作都进行持久化，或者no表示不进行持久化而是借助操作系统的同步将缓存区数据写到磁盘  
appendfsync everysec  
  
\#AOF重写策略（同时满足如下两个策略进行重写）  
\#当AOF文件大小占到初始文件大小的多少百分比时进行重写  
auto-aof-rewrite-percentage 100  
\#触发重写的最小文件大小  
auto-aof-rewrite-min-size 64mb  
  
\#为减少磁盘操作，暂缓重写阶段的磁盘同步  
no-appendfsync-on-rewrite no     
```  

此处的 appendfsync everysec 可以认为是 RDB 和 AOF 的一个折中方案。
 
\#当 bgsave 出错时停止写（MISCONF Redis is configured to save RDB snapshots, but is currently not able to persist on disk.），遇到该错误可以暂时改为 no，当写成功后再改回 yes
stop-writes-on-bgsave-error yes
 
更多 Redis 持久化请参考 [http://redis.readthedocs.org/en/latest/topic/persistence.html](http://redis.readthedocs.org/en/latest/topic/persistence.html)。
 
### Redis 动态调整配置  

获取 maxmemory(10mb)   

Java **代码**    
  
```
127.0.0.1:6660> config get maxmemory  
1) "maxmemory"  
2) "10485760"   
```  
 
设置新的 maxmemory(20mb)  

Java **代码**   
  
```
127.0.0.1:6660> config set maxmemory 20971520  
OK  
```  
  
但是此时重启 redis 后该配置会丢失，可以执行如下命令重写配置文件  

Java **代码**   
  
```
127.0.0.1:6660> config rewrite  
OK   
```  
  
注意：此时所以配置包括主从配置都会重写。
 
### Redis 执行 Lua 脚本  

Redis 客户端支持解析和处理 lua 脚本，因为 Redis 的单线程机制，我们可以借助 Lua 脚本实现一些原子操作，如扣减库存/红包之类的。此处不建议使用 EVAL 直接发送 lua 脚本到客户端，因为其每次都会进行 Lua 脚本的解析，而是使用 SCRIPT LOAD+ EVALSHA 进行操作。未来不知道是否会用 luajit 来代替 lua，让redis lua 脚本性能更强。
 
到此基本的 Redis 知识就讲完了。
 
## Twemproxy 设置   

一旦涉及到一台物理机无法存储的情况就需要考虑使用分片机制将数据存储到多台服务器，可以说是 Redis 集群；如果客户端都是如 Java 没什么问题，但是如果有多种类型客户端（如 PHP、C）等也要使用那么需要保证它们的分片逻辑是一样的；另外随着客户端的增加，连接数也会随之增多，发展到一定地步肯定会出现连接数不够用的；此时 Twemproxy 就可以上场了。主要作用：分片、减少连接数。另外还提供了 Hash Tag 机制来帮助我们将相似的数据存储到同一个分片。另外也可以参考豌豆荚的 [https://github.com/wandoulabs/codis](https://github.com/wandoulabs/codis)。
 
### 基本配置  

其使用 YML 语法，如  

Java 代码   
  
```
server1:  
  listen: 127.0.0.1:1111  
  hash: fnv1a_64  
  distribution: ketama  
  timeout:1000  
  redis: true  
  servers:  
   - 127.0.0.1:6660:1  
   - 127.0.0.1:6661:1    
```  

server1：是给当前分片配置起的名字，一个配置文件可以有多个分片配置；
listen ： 监听的 ip 和端口；
hash：散列算法；
distribution：分片算法，比如一致性 Hash/ 取模；
timeout：连接后端 Redis 或接收响应的超时时间；
redis：是否是 redis 代理，如果是 false 则是 memcached 代理；
servers：代理的服务器列表，该列表会使用 distribution 配置的分片算法进行分片；
 
### 分片算法  

  hash 算法：   
 
    one_at_a_time
    md5
    crc16
    crc32 (crc32 implementation compatible with libmemcached)
    crc32a (correct crc32 implementation as per the spec)
    fnv1_64
    fnv1a_64
    fnv1_32
    fnv1a_32
    hsieh
    murmur
    jenkins  


  分片算法：  
 
    ketama(一致性 Hash 算法)
    modula(取模)
    random(随机算法)

 
### 服务器列表  

  servers:  
   - ip:port:weight alias  
如  
  servers:  
   - 127.0.0.1:6660:1  
   - 127.0.0.1:6661:1  
或者  
  servers:  
   - 127.0.0.1:6660:1 server1  
   - 127.0.0.1:6661:1 server2    

推荐使用后一种方式，默认情况下使用 ip:port:weight 进行散列并分片，这样假设服务器宕机换上新的服务器，那么此时得到的散列值就不一样了，因此建议给每个配置起一个别名来保证映射到自己想要的服务器。即如果不使用一致性 Hash 算法来作缓存服务器，而是作持久化存储服务器时就更有必要了（即不存在服务器下线的情况，即使服务器 ip:port 不一样但仍然要得到一样的分片结果）。
 
### HashTag  

比如一个商品有：商品基本信息(p:id:)、商品介绍(d:id:)、颜色尺码(c:id:)等，假设我们存储时不采用 HashTag 将会导致这些数据不会存储到一个分片，而是分散到多个分片，这样获取时将需要从多个分片获取数据进行合并，无法进行 mget；那么如果有了 HashTag，那么可以使用“::”中间的数据做分片逻辑，这样 id 一样的将会分到一个分片。
 
### nutcracker.yml配置如下 

Java **代码**  
  
```
server1:  
  listen: 127.0.0.1:1111  
  hash: fnv1a_64  
  distribution: ketama  
  redis: true  
  hash_tag: "::"  
  servers:  
   - 127.0.0.1:6660:1 server1  
   - 127.0.0.1:6661:1 server2  
```  
 
### 连接 Twemproxy

Java 代码  
  
```
/usr/servers/redis-2.8.19/src/redis-cli  -p 1111  
```  
  
Java **代码**   
  
```
127.0.0.1:1111> set p:12: 1  
OK  
127.0.0.1:1111> set d:12: 1  
OK  
127.0.0.1:1111> set c:12: 1  
OK  
```  
 
### 在我的服务器上可以连接 6660 端口   

Java **代码**  
  
```
/usr/servers/redis-2.8.19/src/redis-cli  -p 6660  
127.0.0.1:6660> get p:12:   
"1"  
127.0.0.1:6660> get d:12:   
"1"  
127.0.0.1:6660> get c:12:   
"1"  
```  
 
### 一致性 Hash 与服务器宕机  
 
如果我们把 Redis 服务器作为缓存服务器并使用一致性 Hash 进行分片，当有服务器宕机时需要自动从一致性 Hash 环上摘掉，或者其上线后自动加上，此时就需要如下配置：
 
\#是否在节点故障无法响应时自动摘除该节点，如果作为存储需要设置为为false
auto\_eject_hosts: true
\#重试时间（毫秒），重新连接一个临时摘掉的故障节点的间隔，如果判断节点正常会自动加到一致性Hash环上
server\_retry\_timeout: 30000
\#节点故障无法响应多少次从一致性Hash环临时摘掉它，默认是2
server\_failure\_limit: 2
 
### 支持的 Redis 命令  

不是所有 Redis 命令都支持，请参考 [https://github.com/twitter/twemproxy/blob/master/notes/redis.md](https://github.com/twitter/twemproxy/blob/master/notes/redis.md)。
 
因为我们所有的 Twemproxy 配置文件规则都是一样的，因此我们应该将其移到我们项目中。  

Java **代码**   
  
```
cp /usr/servers/twemproxy-0.4.0/conf/nutcracker.yml  /usr/example/   
```  
 
另外 Twemproxy 提供了启动/重启/停止脚本方便操作，但是需要修改配置文件位置为 /usr/example/nutcracker.yml。  

Java **代码**  
  
```
chmod +x /usr/servers/twemproxy-0.4.0/scripts/nutcracker.init   
vim /usr/servers/twemproxy-0.4.0/scripts/nutcracker.init    
```  
 
将 OPTIONS 改为  

OPTIONS="-d -c /usr/example/nutcracker.yml"
 
另外注释掉. /etc/rc.d/init.d/functions；将 daemon --user ${USER} ${prog} $OPTIONS 改为 ${prog} $OPTIONS；将 killproc 改为 killall。
 
这样就可以使用如下脚本进行启动、重启、停止了。  
/usr/servers/twemproxy-0.4.0/scripts/nutcracker.init {start|stop|status|restart|reload|condrestart}
 
对于扩容最简单的办法是：  
1、 创建新的集群；  
2、 双写两个集群；  
3、 把数据从老集群迁移到新集群（不存在才设置值，防止覆盖新的值）；  
4、 复制速度要根据实际情况调整，不能影响老集群的性能；  
5、 切换到新集群即可，如果使用Twemproxy代理层的话，可以做到迁移对读的应用透明。  
 