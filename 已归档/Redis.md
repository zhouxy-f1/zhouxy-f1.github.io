## 第一章 Redis理论基础

##### 应用场景

```bash
1.缓存-键过期时间
    缓存session会话
    缓存用户信息,找不到再去mysql查,查到然后回写到redis
    商城优惠卷过期时间
    
2.排行榜-列表&有序集合
    热度排名排行榜
    发布时间排行榜
    
3.计数器应用-天然支持计数器
    帖子浏览数
    视频播放次数
    商品浏览数
    点赞/点踩
    
4.社交网络-集合
    踩/赞,粉丝,共同好友/喜好,推送,打标签
    
5.消息队列系统-发布订阅
    配合elk实现日志收集
```

##### 软件对比

```bash
#redis优势
1.速度快：所有的数据都存放在内存中；使用c语言实现；使用单线程架构
2.支持5种数据结构:字符串,哈希,列表,集合,有序集合，地理位置
3.功能多：提供了键过期功能,可以实现缓存；提供了发布订阅功能,可以实现消息系统；提供了pipeline功能,客户端可以将一批命令一次性传到 Redis,减少了网络开销
4.简单稳定：源码很少,3.0版本以后5万行左右；使用单线程模型法,是的Redis服务端处理模型变得简单；不依赖操作系统的中的类库
5.客户端语言多：java,PHP,python,C,C++,Nodejs等
6.数据持久化：RDB和AOF
7.原生支持主从复制、高可用、集群

#对比
memcached: 纯内存，性能高，但无持久化，一旦故障缓存穿透会致命，安全性不高数据类型单一，无法直接上生产，适合大厂二次开发，比如淘宝开发出Tair。多核结构，适合大量用户少量读写

redis: 数据类型丰富，支持持久化和事务，原生支持高可用，原生支持分布式集群。单核工作模式，单线程读写性能极高，适合少量用户，大量读写，所以生产中用单机多实例，比如京东、新浪、直播、网页游戏等
```

## 第二章 Redis安装部署

##### 方式1：通过yum

```bash
yum install -y http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
yum --enablerepo=remi list redis --showduplicates
yum --enablerepo=remi install redis

systemctl start redis
systemctl enable redis
systemctl status redis
```

##### 方式2:编译安装

```bash




```

##### 安装部署

```mysql
#目录及配置一览
/home/redis/6379							#主程序目录，需要手动创建
/etc/redis/6379.conf					#配置文件，需要在编译安装完成后手动创建
/var/log/redis_6379.log				#日志文件，根据配置文件自动创建，路径配置成这样是为了跟redis_init_scrip里一致
/var/run/redis_6379.pid				#pid文件，启动redis时自动生成
/etc/rc.d/init.d/redis_6379		#从源码包拷贝过来的，用于通过init启停redis
/usr/lib/systemd/system/redis_6379.service	#手动创建的，用于通过systemd启停redis

#安装
yum install -y gcc automake autoconf libtool make			#如果没有C环境，需要提前安装
curl -O http://download.redis.io/releases/redis-5.0.8.tar.gz
tar xf redis-5.0.8.tar.gz -C /opt
ln -s /opt/redis-5.0.8/ /opt/redis
cd /opt/redis
make && make install											#make是生成可执行文件，可用make clean清除

//处理编译报错
报错1：make[3]: gcc: Command not found
=>	yum install -y gcc

报错2：cc: error: ../deps/hiredis/libhiredis.a: No such file or directory
=>	cd deps && make lua hiredis linenoise

报错3：zmalloc.h:50:31: fatal error: jemalloc/jemalloc.h: No such file or directory
=>	make MALLOC=libc

报错4：*** No rule to make target '../deps/jemalloc/include/jemalloc/jemalloc.h', needed by 'adlist.o'.  Stop.
=>	make MALLOC=libc

#配置
mkdir /etc/redis /home/redis/6379				#启动时不会自动创建
cat>/etc/redis/6379.conf<<EOF
daemonize yes
bind 127.0.0.1 172.16.1.186
port 6379
dir /home/redis/6379
pidfile /var/run/redis_6379.pid
logfile /var/log/redis_6379.log
databases 16
requirepass test
EOF

#原始启停redis
redis-server /etc/redis/6379.conf					#启动服务
redis-cli -h 172.16.1.186 -p 6379 				#远程连接数据库
redis-cli																	#本地连接数据库
shutdown																 	#停止服务,需要先登陆并验证成功后再shutdown
```

##### 两种启停方式

```bash
#第1种：使用init启停redis
cp /opt/redis/utils/redis_init_script  /etc/init.d/redis_6379
改第35行为：$CLIEXEC -h 172.16.1.186 -p $REDISPORT -a test shutdown

/etc/init.d/redis_6379 start				#启动
/etc/init.d/redis_6379 stop					#停止


#第2种：使用systemd启停redis
cat >/usr/lib/systemd/system/redis_6379.service<<EOF
[Unit]
Description=Redis persistent key-value database
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/redis-server  /etc/redis/6379.conf --supervised systemd
ExecStop=/usr/local/bin/redis-cli -h 172.16.1.186  -p 6379 shutdown
Type=forking

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload


systemctl start redis_6379			#启动redis
systemctl status redis_6379			#查看redis
systemctl stop redis_6379				#停止redis




#连接配置
daemonize yes
bind 127.0.0.1 192.168.50.126
port 6379
dir /home/redis/
pidfile /var/run/redis.pid
logfile /var/log/redis.log
databases 16
requirepass test


#开启RDB自动触发持久化
save 900 1              #900秒内发生1次修改触发RDB
save 300 10             #300秒内发生10次修改触发
save 60 10000           #60秒内发生10000次修改触发

#AOF持久化（每秒触发一次）
#appendonly     yes
#appendfilename  redis.aof
#appendfsync    everysec
```

##### 安全认证

```bash
redis的安全认证仅是一个密码，没有用户、角色、权限等配置，但redis只会放在内网运行，相对还是很安全的

#立即生效，退出登陆时需要新密码，重启服务失效
127.0.0.1:6379>config set requirepass 123

#配置文件追加：requirepass 123，重启生效
cat >>/data/redis_6379/conf/redis.conf<<EOF
requirepass 123
EOF

#认证
redis-cli  -a 123														//登陆时认证
127.0.0.1:6379> auth 123										//登陆后在命令行认证
```

## 第三章 redis管理命令

```bash
#查看redis状态信息
info												//查看以下所有信息
info server									//服务端信息	
info clients								//客户端信息
info memory									//内存信息
info persistence						//持久化信息
info stats									//统计信息
info replication						//主从复制信息
info cpu										//cpu信息
info cluster								//集群信息
info keyspace								//库相关统计信息

#查看客户端连接信息
client list									//有几个会话就有几条信息
client kill 10.0.0.52:50742	//杀掉某个会话


#查看配置信息
config get *								//查看所有配置信息
config get dir
config get dbfilename
config get maxmemory				//查看内存配置
config set maxmemory 60G		//在线设置最大内存
config resetstat						//重置所有配置

#对库的操作
flushdb											//清空当前库（生产中勿用）
flushall										//清空所有库（生产中勿用）
select [0..15]							//选择要操作的库，redis共16个库，默认在0号库操作，库与库之间相互隔离
dbsize											//查看当前库共有多少个key
keys *											//查看所有键（生产中勿用）
			
			
#对数据的操作			
set k1 v1										//定义键的值
exists k1										//查看键是否存在
type k1											//查看键所存储的类型
rename k1 key1							//变更键名
del key1										//删除键
expire|pexpire	k1 10				//设定name键的生存时间（秒|毫秒）
ttl|pttl	k1								//查看name键剩余生存时间（秒|毫秒）
persist k1									//取消name键生存时间设置
			
			
#实时监控输入的命令			
monitor											//无法监控命令的回显信息
			
#关闭redis服务			
shutdown										//关闭服务
```

## 第四章 数据类型

##### 数据类型

```bash
#介绍
string		 :字符类型  例 zhang3
hash			 :字典类型  例 id:101 name:li4 age:18 gender:m
list			 :列表     例 [...,day3,day2,day1]
set				 :集合	   例（zhao,qian,sun,li...）
sorted set :有序集合  例（歌曲1，歌曲2，歌曲3）
```

##### 使用场景

```bash
#string应用场景
1.session共享:比如保存session时长一天，24小时后需要再次登陆
2.常规计数：粉丝数、点赞数、订阅数、评论数等计数
incr num   			//每次加1
decr num   			//每次减1
incrby num 100  //每次加100
decrby num 100  //每次减100
get num					//查看当前关注数

#hash类型应用场景
1.做mysql数据库缓存：由应用程序从mysql拿数据，经转换后灌入redis;或者人为导出再灌入
2.存储部分经常变更的数据

重点在于给本组数据取名，即 表名:主键名
hmset user:1 name zhou age 18						//生成hash类型数据
hmset user:1 name												//查看一个值
hmset user:1 name age										//查看多个值
hgetall user:1													//查看所有值

把mysql数据库导出为kv对形式，再灌入redis
select concat("hmset city : ",id," id ",id," name ",name," countrycode ",countrycode," district ",district," population ",population) from city limit 10 into outfile '/tmp/hmset.txt'	//拼命令导出
cat /tmp/hmset.txt | redis-cli -a 123																										//导入redis


#list类型应用场景
1.消息列队系统：比如新浪微博把前5000条微博放redis常驻缓存，更早的数据存到mysql
2.朋友圈3天可见：

lpush list1 A B C 							//依次存入CBA
rpush list1 D E F								//依次存入DEF
lrange list1 0 -1								//查看所有存入的数据
llen list1											//查看列表长度
lpop list1											//删最左的一个数据
rpop list1											//删最右的一个数据
del list1												//删除键
exists list1										//查键是否存在

举例：
lpush wechat "today is 1"
lpush wechat "today is 2"
lpush wechat "today is 3"
lpush wechat "today is 4"
lpush wechat "today is 5"
lrange wechat 0 -1
lrange wechat 0 2	


#set集合类型
1.QQ中的共同好友、可能认识的人，微博中共同关注、二度好友等

集合中不允许出现重复的值，重复SADD set1，不会覆盖原成员，只会增加成员，自动过滤重复的成员

sadd set1 1 3 5 6 7		//新建集合set1
sadd set2 1 2 5 7 9		//新建集合set2
scard set1						//查看集合成员数量
smembers set1					//查看集合成员
srem set1 7						//删除集合中某个成员
sismember set1 7			//查看某个成员是否在集合中
sinter set1 set2			//交集
sunion set1 set2			//并集
sdiff set1 set2				//差集，set1中set2没有的部分
sdiff set2 set1				//差集，set2中set1没有的部分


#sorted set有序集合
主要用于割取排行榜
```

##### 发布订阅

```bash
SUBSCRIBE fm1039				//先订阅1个或多个频道
UNSUBSCRIBE fm1039 			//取消订阅，不指定哪个频道则退订所有
PSUBSCRIBE  fm*					//订阅某规则的频道
PUNSUBSCRIBE fm* 				//取消订阅符合规则的频道，不指定则退订所有规则

PUBLISH fm1039 "it's a fine day today" 	//其它窗口发布信息
```

##### 事务

```bash
redis事务的实现不同与mysql，redis开启事务后把所有任务都放入队列中，并不执行
而mysql开启事务后，执行的语句都在内存中执行完，commit只是刷到磁盘，rollback时用undo回滚

MULTI					//开启事务
set a 1				//把set a 1 放入队列中，不执行
set a 2				//仅放入队列
EXEC 					//执行
DISCARD				//把事务从队列里丢弃


#乐观锁
set ticket 1
watch ticket 		//监控键值对，使无法同时更新
multi
decr ticket

```

##### 禁用危险命令

```mysql
#禁用危险命令
cat >>/data/redis_6379/conf/redis.conf<<EOF
rename-command KEYS ""
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command CONFIG ""
EOF

还可以给自己留后门
cat >>/data/redis_6379/conf/redis.conf<<EOF
rename-command KEYS "KS"
rename-command FLUSHALL "FL"
rename-command FLUSHDB "FD"
rename-command CONFIG "CFG"
EOF

#在线修改配置，不用重启
127.0.0.1:6379> CONFIG GET  *
127.0.0.1:6379> CONFIG GET  bind
127.0.0.1:6379> CONFIG GET  a*
127.0.0.1:6379> CONFIG SET requirepass 1234				//立即生效，退出再登陆需要新密码，重启失效
```

## 第五章 持久化

##### RDB与AOF原理

```bash
#原理
RDB持久化 (snapshotting)：是把当前进程数据生成快照保存到磁盘中/data/redis_6379/dump.rdb文件的过程，由是RDB是压缩的二进制文件，仅保存当时的数据状态，速度很快，非常适合做全量备份，可写入定时任务，比如6小时执行一次bgsave做全备，把备份拷贝到远程主机

AOF(append only file)持久化:以追加独立日志的方式记录每次修改操作,可以最大限度保证数据不丢，类似于mysql的binlog，重启时再执行AOF文件中的命令达到恢复数据的目的.AOF的主要作用是解决了数据持久化的实时性,目前已经是Redis持久化的主流方式


#RDB和AOF优缺点
RDB: 快照,把当前内存里的状态快照到磁盘上
优点: 压缩格式/恢复速度快
缺点: 可能会丢失将近1分钟的数据

AOF: 类似于mysql的binlog，重写，每次操作都写一次/1秒写一次
优点: 安全,有可能会丢失1秒的数据
缺点: 文件比较大,恢复速度慢
```

##### 配置方法

```bash
#配置RDB
cat >>/data/redis_6379/conf/redis.conf<<EOF
dbfilename redis.rdb														//指定本地持久化文件名（开启手动触发RDB）
save 900 1																			//15分钟内发生一次修改（开启自动触发条件）
save 300 10																			//5分钟内发生10次修改
save 60 10000																		//1分钟内发生10000次修改
EOF


提问：当执行redis-cli shutdown 或 kill pkill killall等操作时，redis会不会执行RDB持久化？
答：如果配置了save参数时，以上操作会触发自动持久化，等内部执行完bgsave再shutdown，kill -9 除外
	 如果没有配置save参数，则不会执行RDB，数据会丢


提问：备份的RDB文件怎么恢复到redis
答：需要恢复备份的RDB文件时，需要停redis服务，再拷贝RDB文件到redis配置文件中指定的目录，名称要一致，再启动即可
如果不停服务，直接替换，则会找不到句柄，无法成功恢复
如果把全备拷贝回来后再重启，则在停止服务时，就会覆盖掉全备的文件；所以必须先停止服务，再拷贝全备文件，再启动

#配置AOF
cat >>/data/redis_6379/conf/redis.conf<<EOF
appendonly yes																//开启AOF持久化
appendfilename  redis.aof 										//指定本地AOF持久化文件名
appendfsync everysec													//指定AOF频率，always(每步),官方建议everysec(每秒)
EOF

#提问：如果AOF和RDB文件同时存在,redis会如何读取?
答：优先读取AOF文件
```

##### 补充点

```bash
#补充点
1.重写机制
	随着命令不断写入AOF,文件会越来越大,为了解决这个问题,Redis引入了AOF重写机制压缩文件体积。AOF文件重写是把Redis进程内的数据转化为写命令同步到新AOF文件的过程.
手动触发重写机制:直接调用bgrewriteaof命令


2.手动触发RDB方式
1）save命令：会阻塞当前Redis服务器,直到RDB过程完成为止,对于内存比较大的实例会造成长时间阻塞,不建议使用.
2）bgsave命令：redis父进程fork出子进程,由子进程完成RDB后自动结束.阻塞只发生在fork阶段,一般时间很短.

3.自动触发RDB的情况
1）配置了save参数
2）执行主从复制，主库自动bgsave
3）执行debug reload命令

#写定时任务做全备
```

## 第六章 主复制集

##### 原理

```bash
构建主从关系只需要在副本上执行slaveof 10.0.0.15 6379 即可完成，具体工作流程如下
1.从库执行slaveof立即发送sync信号给主库
2.主库收到sync信号后，自动触发bgsave，保存RDB快照到磁盘，然后发送给从库
3.从库收到RDB后先清空自己的数据，再把接收到的rdb加载到内存，至此主从数据一致
4.主库把在构建主从期间产生的新数据发给从库，主从复制集正常工作
5.此后主库以广播的形式主动发送新数据给从库
6.如果从库宕机，再次连接上时，从库读取配置文件重新构建主从
```

##### 搭建主从

```bash
#准备多实例(主: 6379  从: 6380、6381)

cp -a /data/redis_6379 /data/redis_6380
cp -a /data/redis_6379 /data/redis_6381
sed -i s#6379#6380#g /data/redis_6380/conf/redis.conf
sed -i s#6379#6380#g /data/redis_6381/conf/redis.conf
sed -i '$imasterauth 123'  /data/redis_6380/conf/redis.conf							//$指末行，a是增加，加主库认证
sed -i '$imasterauth 123'  /data/redis_6381/conf/redis.conf							//主从密码统一，否则哨兵无法工作

redis-server /data/redis_6379/conf/redis.conf
redis-server /data/redis_6380/conf/redis.conf
redis-server /data/redis_6381/conf/redis.conf

#开启主从
redis-cli -p 6380 -a 123 slaveof 127.0.0.1 6379
redis-cli -p 6381 -a 123 slaveof 127.0.0.1 6379


#切换主从命令
slaveof 10.0.0.53 6379														//切换db03为主库
slaveof no one																		//断开主从
info replication																	//查看主从状态

#保障主从数据一致
min-slaves-to-write 1
min-slaves-max-lag  3

#注意事项
1.搭建主从复制时，从库会清空自己原有的数据，不要弄错主库，否则会丢数据
2.强烈建议在主库开启持久化，因为主库一旦宕机，数据全丢，重启时从库依然复制主库，从库就也清空了。主库宕机不能重启
3.从节点默认只读不可写
4.主从复制故障转移需要人工介入
	-修改代码指向redis的IP地址
	-在从节点执行slaveof no one，终止主从关系
```

## 第七章 高可用

##### Sentinel原理

```bash
哨兵本身也是一台redis服务器，可以管理几百套集群，哨兵之间通过发布订阅同一频道发现彼此，通过给各节点发info命令获取状态

哨兵主要工作：
1.监控各节点状态
2.自动选主，切换
3.从库指向新主库
4.应用透明: 应用程序指向哨兵的ip和端口，再路由到各节点
5.自动处理故障节点

#
配置哨兵时为一般为单数
```

##### 配置

```bash
mkdir sentinel_26379/{conf,logs,pid} -p

cat >/data/sentinel_26379/conf/sentinel.conf <<EOF
port 26379
dir "/data/sentinel_26379"
sentinel monitor mymaster 127.0.0.1 6379 1						//监控的主节点，判断主节点失败需要1个哨兵节点同意
sentinel down-after-milliseconds mymaster 5000				//哨兵3秒检测不到节点就判断节点断线
sentinel auth-pass mymaster 123 											//哨兵认证密码
EOF

redis-sentinel  /data/sentinel_26379/conf/sentinel.conf &>/tmp/sentinel.log &				//后台启动哨兵

#测试
tailf /tmp/sentinel.log															//打开实时日志
redis-cli -p 6379 -a 123 shutdown										//关闭主库: 选举6380为新主，把6281指向新主
redis-server /data/redis_6379/conf/redis.conf				//修复原主库: 自动加入集群并指向新主

#哨兵节点管理命令
redis-cli -h 10.0.0.51 -p 26379																				//登陆哨兵节点
127.0.0.1:26379> Info Sentinel																				//查看哨兵状态
127.0.0.1:26379> Sentinel get-master-addr-by-name <master name> 			//查看当前主库的IP
127.0.0.1:26379> sentinel failover myredis														//主动故障转移
127.0.0.1:26379> CONFIG SET slave-priority 0													//配置权重
127.0.0.1:26379> Sentinel flushconfig																	//重置哨兵配置
```

## 第八章 redis-cluster

##### 集群简介

```bash
#去中心化思想
redis-cluster没有中心节点的说法，使用hash solt方式将16384个slot平均分配到所有节点。假如有3个节点，那么A节点分配0~5500哈希槽；B节点分配5501~11000;C节点分配11001~16383的哈希槽。
存储数据时：把key拿来做CRC16运算，得到一个数再取模，得到一个0~16384间的数字，这个数字就是该key的值存放的槽位
取数据时：用get key来取值，做key再做CRC16再取模，找到槽位号再取数据


#自带高可用
每个节点内部采用哨兵模式，1主1从。如果某个节点的主库坏了，则将其从库提升为主，如果从库环了，不影响集群，修复后自动加入集群。整个集群损坏一个节点，不影响；损坏2个节点则集群slot映射不完整，集群进入fail状态

注：不要把同一对主同放同一台物理机上
```

![image-20200321225637920](https://tva1.sinaimg.cn/large/00831rSTgy1gd1yau2t7aj30p206tab0.jpg)

##### 部署集群

```bash
#准备3台服务器，起6个实例。在db01用7000和7001端口起2个实例，db02和03如下
db01		10.0.0.51		172.16.1.51		7000|7001
db02		10.0.0.52		172.16.1.52		7002|7003
db03		10.0.0.53		172.16.1.53		7004|7005

实例目录结构如下
├── cluster_7000
│   ├── conf
│   │   └── redis.conf
│   ├── logs
│   └── pid
#db01配置文件,并启动服务
mkdir /data/700{0,1} -p
cat >/data/7000/redis.conf<<EOF
port 7000
daemonize yes
pidfile /data/7000/redis.pid
loglevel notice
logfile "/data/7000/redis.log"
dbfilename dump.rdb
dir /data/7000
protected-mode no
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
EOF
cat >/data/7001/redis.conf<<EOF
port 7001
daemonize yes
pidfile /data/7001/redis.pid
loglevel notice
logfile "/data/7001/redis.log"
dbfilename dump.rdb
dir /data/7001
protected-mode no
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
EOF
redis-server /data/7000/redis.conf
redis-server /data/7001/redis.conf
ss -lntup |grep redis


#db02
mkdir /data/700{2,3} -p
cat >/data/7002/redis.conf<<EOF
port 7002
daemonize yes
pidfile /data/7002/redis.pid
loglevel notice
logfile "/data/7002/redis.log"
dbfilename dump.rdb
dir /data/7002
protected-mode no
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
EOF
cat >/data/7003/redis.conf<<EOF
port 7003
daemonize yes
pidfile /data/7003/redis.pid
loglevel notice
logfile "/data/7003/redis.log"
dbfilename dump.rdb
dir /data/7003
protected-mode no
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
EOF
redis-server /data/7002/redis.conf
redis-server /data/7003/redis.conf
ss -lntup |grep redis


#db03
mkdir /data/700{4,5} -p
cat >/data/7004/redis.conf<<EOF
port 7004
daemonize yes
pidfile /data/7004/redis.pid
loglevel notice
logfile "/data/7004/redis.log"
dbfilename dump.rdb
dir /data/7004
protected-mode no
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
EOF
cat >/data/7005/redis.conf<<EOF
port 7005
daemonize yes
pidfile /data/7005/redis.pid
loglevel notice
logfile "/data/7005/redis.log"
dbfilename dump.rdb
dir /data/7005
protected-mode no
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
EOF
redis-server /data/7004/redis.conf
redis-server /data/7005/redis.conf
ss -lntup |grep redis

#安装集群插件，让ruby可以操作redis
yum install -y ruby rubygems
gem sources -a http://mirrors.aliyun.com/rubygems/
gem sources  --remove https://rubygems.org/
gem sources -l
gem install redis

#创建集群
/app/redis/src/redis-trib.rb create --replicas 1 172.16.1.51:7000 172.16.1.52:7002 172.16.1.53:7004 172.16.1.52:7003  172.16.1.53:7005 172.16.1.51:7001

设命令中主机顺序为：A B C A1 B1 C1
则：默认ABC为主库，A1是A的从库，A为集群管理节点，端口7000
--replicas 1 中1代表每个master后有1个slave


/app/redis/src/redis-trib.rb create --replicas 1 192.168.1.12:7000 192.168.1.13:7002 192.168.1.14:7004 192.168.1.13:7003  192.168.1.14:7005 192.168.1.12:7001
```

##### 检查集群健康状态

```bash
/app/redis/src/redis-trib.rb check 10.0.0.51:7000						//是否完整
/app/redis/src/redis-trib.rb rebalance 10.0.0.51:7000				//槽位数误差是否小于2%
redis-cli -h 10.0.0.53 -p 6381 CLUSTER REPLICATE 51-6390的ID	//
redis-cli -h 10.0.0.51 -p 6391 CLUSTER REPLICATE 51-6380的ID //
```

##### 增删节点

```bash
增加节点：准备实例-主节点加入集群-重新分配slot-加入从节点
删除节点：分三次把槽位原路返回--删除主节点-删除从节点

#增加节点
1.先准备实例 172.16.1.54:7006 和 172.16.1.54:7007，然后启动这两个实例
2.再把主节点加入集群
	/app/redis/src/redis-trib.rb add-node 172.16.1.54:7006 172.16.1.51:7000			//后面是集群管理节点
3.给新加入节点分配槽位
	/app/redis/src/redis-trib.rb reshard 127.0.0.1:7000													//需移动(16384/4)个槽位
4.最后把从节点加入集群
	/app/redis/src/redis-trib.rb add-node --slave --master-id e1948fe69a49e1b9cbf6e3fc873b801ad1dadc0f 172.16.1.54:7007 172.16.1.51:7000		//7000是管理节点																  
 
#删除节点
1.先要分3次把槽位返还，再删除主库和从库
2./app/redis/src/redis-trib.rb reshard 127.0.0.1:7000
	/app/redis/src/redis-trib.rb del-node 172.16.1.54:7006 e1948fe69a49e1b9cbf6e3fc873b801ad1dadc0f
	/app/redis/src/redis-trib.rb del-node 172.16.1.54:7007 9651b990497f176c751e8156aaa96956ecfbe99f
```

## 第九章 API支持

##### 安装环境

```bash
python为例
yum install -y python36 
python3 -V
yum install -y python36-pip
pip3 install redis 
pip3 install redis-py-cluster

++++++++++++源码方式+++++++++++++++
https://redis.io/clients
下载redis-py-master.zip
安装驱动：
unzip redis-py-master.zip
cd redis-py-master
python3 setup.py install

redis cluster的连接并操作（python2.7.2以上版本才支持redis cluster，我们选择的是3.6）
https://github.com/Grokzen/redis-py-cluster
安装redis-cluser的客户端程序
cd redis-py-cluster-unstable
python3 setup.py install
+++++++++++++++++++++++++++++++++
```

##### 测试

```bash
#单机实例
[root@db01 ~]# redis-server /data/6379/redis.conf 

python3
>>>import redis
>>>r = redis.StrictRedis(host='10.0.0.51', port=6379, db=0,password='123456')
>>>r.set('oldboy', 'oldguo')
>>>r.get('oldboy')

#哨兵集群
[root@db01 ~]# redis-server /data/6380/redis.conf
[root@db01 ~]# redis-server /data/6381/redis.conf
[root@db01 ~]# redis-server /data/6382/redis.conf 
[root@db01 ~]# redis-sentinel /data/26380/sentinel.conf &
--------------------------------
 导入redis sentinel包
>>>from redis.sentinel import Sentinel  
指定sentinel的地址和端口号
>>> sentinel = Sentinel([('localhost', 26380)], socket_timeout=0.1)  
测试，获取以下主库和从库的信息
>>> sentinel.discover_master('mymaster')  
>>> sentinel.discover_slaves('mymaster')  
#哨兵读写分离
写节点
>>> master = sentinel.master_for('mymaster', socket_timeout=0.1,password="123")  
读节点
>>> slave = sentinel.slave_for('mymaster', socket_timeout=0.1,password="123")  
读写分离测试   key     
>>> master.set('oldboy', '123')  
>>> slave.get('oldboy')  

#redis-cluster集群
python3
>>> from rediscluster import StrictRedisCluster  
>>> startup_nodes = [{"host":"127.0.0.1", "port": "7000"},{"host":"127.0.0.1", "port": "7001"},{"host":"127.0.0.1", "port": "7002"}]  
Note: decode_responses must be set to True when used with python3  
>>> rc = StrictRedisCluster(startup_nodes=startup_nodes, decode_responses=True)  
>>> rc.set("foo", "bar")  
True  
>>> print(rc.get("foo"))  
'bar'
```

##### 缓存相关概念

```bash
#缓存穿透
访问一个不存在的key，缓存不起作用，请求会穿透到DB，流量大时DB会挂掉。
解决方案:
采用布隆过滤器，使用一个足够大的bitmap，用于存储可能访问的key，不存在的key直接被过滤；
访问key未在DB查询到值，也将空值写进缓存，但可以设置较短过期时间。

#缓存雪崩
大量的key设置了相同的过期时间，导致在缓存在同一时刻全部失效，造成瞬时DB请求量大、压力骤增，引起雪崩。
解决方案:
可以给缓存设置过期时间时加上一个随机值时间，使得每个key的过期时间分布开来，不会集中在同一时刻失效。

#缓存击穿
一个存在的key，在缓存过期的一刻，同时有大量的请求，这些请求都会击穿到DB，造成瞬时DB请求量大、压力骤增。
解决方案:
在访问key之前，采用SETNX（set if not exists）来设置另一个短期key来锁住当前key的访问，访问结束再删除该短期key。
```

## 第十章 数据迁移

```bash
#安装工具
yum install libtool autoconf automake git bzip2 -y 
cd /opt/
git clone https://github.com/vipshop/redis-migrate-tool.git
cd redis-migrate-tool/
autoreconf -fvi
./configure
make && make install 

#单节点迁移到集群
cat > 6379_to_6380.conf << EOF																		#配置文件
[source]
type: single
servers:
- 10.0.0.51:6379

[target]
type: redis cluster
servers:
- 10.0.0.51:6380 

[common]
listen: 0.0.0.0:8888
source_safe: true
EOF

redis-server /opt/redis_6379/conf/redis_6379.conf 												#生成测试数据
cat >input_6379.sh<<EOF 
#!/bin/bash
for i in {1..1000}
do
    redis-cli -c -h db01 -p 6379 set oldzhang_\${i} oldzhang_\${i}
    echo "set oldzhang_\${i} is ok"
done
EOF

redis-migrate-tool -c 6379_to_6380.conf										#迁移
redis-migrate-tool -c 6379_to_6380.conf -C redis_check		#检查结果


#RDB文件迁移到集群
redis-cli -h db01 -p 6381 BGSAVE													#收集rdb文件
redis-cli -h db02 -p 6381 BGSAVE
redis-cli -h db03 -p 6381 BGSAVE

mkdir rdb_backup																				#rdb文件拉过来
cd rdb_backup/
scp db01:/data/redis_6381/redis_6381.rdb db01_6381.rdb
scp db02:/data/redis_6381/redis_6381.rdb db02_6381.rdb
scp db03:/data/redis_6381/redis_6381.rdb db03_6381.rdb

redis-cli -c -h db01 -p 6380 flushall												#清空数据
redis-cli -c -h db02 -p 6380 flushall
redis-cli -c -h db03 -p 6380 flushall	

cat >rdb_to_cluter.conf <<EOF																#配置
[source]
type: rdb file
servers:
- /root/rdb_backup/db01_6381.rdb 
- /root/rdb_backup/db02_6381.rdb 
- /root/rdb_backup/db03_6381.rdb 

[target]
type: redis cluster
servers:
- 10.0.0.51:6380 

[common]
listen: 0.0.0.0:8888
source_safe: true
EOF

redis-migrate-tool -c rdb_to_cluter.conf 													#导入数据
```

## 第十一章 分析key大小

```bash
需求背景
redis的内存使用太大键值太多,不知道哪些键值占用的容量比较大,而且在线分析会影响性能.
1.安装命令
yum install python-pip gcc python-devel -y
cd /opt/
git clone https://github.com/sripathikrishnan/redis-rdb-tools
cd redis-rdb-tools
pip install python-lzf
python setup.py install
2.生成测试数据
redis-cli -h db01 -p 6379 set txt $(cat txt.txt)
3.执行bgsave生成rdb文件
redis-cli -h db01 -p 6379 BGSAVE
4.使用工具分析
cd /data/redis_6379/
rdb -c memory redis_6379.rdb -f redis_6379.rdb.csv
5.过滤分析
awk -F"," '{print $4,$3}' redis_6379.rdb.csv |sort -r
6.将结果整理汇报给领导,询问开发这个key是否可以删除
```

## 第十二章 内存管理

```bash
1.设置内存最大限制
config set maxmemory 2G
2.内存回收机制
当达到内存使用限制之后redis会出发对应的控制策略
redis支持6种策略：
	1).noevicition       默认策略，不会删除任务数据，拒绝所有写入操作并返回客户端错误信息，此时只响应读操作
	2).volatile-lru      根据LRU算法删除设置了超时属性的key，指导腾出足够空间为止，如果没有可删除的key，则退回到noevicition策略
	3).allkeys-lru       根据LRU算法删除key，不管数据有没有设置超时属性
	4).allkeys-random    随机删除所有key
	5).volatile-random   随机删除过期key
	5).volatile-ttl      根据key的ttl，删除最近要过期的key
3.动态配置
config set maxmemory-policy
```













































