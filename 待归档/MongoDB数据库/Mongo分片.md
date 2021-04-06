# 第1章 分片的概念

```
1.有了副本集，为什么还需要分片？
副本集资源利用率不高
分片可以提高资源利用率

2.分片的缺点
理想情况下需要机器比较多
配置和运维变得复杂且困难
提前规划好特别重要，一旦建立后在想改变架构变得很困难
```

# 第2章 分片工作流程

```
1.路由服务mongos 
路由服务，提供代理，替用户去向后请求shard分片的数据

2.数据节点shard
负责处理数据的节点，每个shard都是分片集群的一部分

3.分片配置信息服务器config 
保存数据分配在哪个shard上
保存所有shard的配置信
提供给mongos查询服务

4.片键
数据存放到哪个shard的区分规则
片键就是索引
选择片键的依据：
能够被经常访问到的字段
索引字段基数够大
```

# 第3章 分片的分类

## 1.区间片键

```
id  name   host  sex 

1   zhang  SH    boy
2   ya     BJ    boy 
3   yaya   SZ    girl
```

如果以id作为片键：

```
id
1-100   shard1
100-200 shard2
200-300 shard3 
300-+无穷 shard4
```

如果以host作为片键：

```
SH  shard1
BJ  shard2 
SZ  shard3 
```

## 2.hash片键:

特点：足够平均，足够随机

```
id  name   host  sex 
1     zhang  SH    boy
2   ya     BJ    boy 
3   yaya   SZ    girl
```

如果以id作为片键：

索引：id

```
1      hash 之后 shard2 
2    hash 之后 shard3 
3    hash 之后 shard1 
```

# 第4章 IP端口目录规划

## 1.IP端口规划

```
db01    10.0.0.51   
            Shard1_Master   28100
            Shard2_Slave    28200
            Shard3_Arbiter  28300
            Config Server   40000
            mongos Server   60000
        
db02    10.0.0.52   
            Shard2_Master   28100
            Shard3_Slave    28200
            Shard1_Arbiter  28300
            Config Server   40000
            mongos Server   60000
        
db03    10.0.0.53   
            Shard3_Master   28100
            Shard1_Slave    28200
            Shard2_Arbiter  28300
            Config Server   40000
            mongos Server   60000
```

## 2.目录规划

服务目录：

```
/opt/master/{conf,log,pid}
/opt/slave//{conf,log,pid}
/opt/arbiter/{conf,log,pid}
/data/config/{conf,log,pid}
/data/mongos/{conf,log,pid}
```

数据目录：

```
/data/master/
/data/slave/
/data/arbiter/
/data/config/
```

# 第5章 分片集群搭建副本集步骤

## 0.整体步骤

```
1.先搭建副本集
2.搭建config副本集
3.搭建mongos副本集
4.数据库启用分片功能
5.集合设置片键
6.插入测试数据
7.检查是否分片
8.安装图形化工具 
```

## 1.安装软件

注意：三台服务器都操作！！！

```
tar xf mongodb-linux-x86_64-3.6.13.tgz -C /opt/
ln -s /opt/mongodb-linux-x86_64-3.6.13 /opt/mongodb
echo 'export PATH=$PATH:/opt/mongodb/bin' >> /etc/profile
source /etc/profile
```

## 2.创建目录

注意：三台服务器都操作！！！

```
mkdir -p /opt/master/{conf,log,pid}
mkdir -p /opt/slave/{conf,log,pid}
mkdir -p /opt/arbiter/{conf,log,pid}

mkdir -p /data/master/
mkdir -p /data/slave/
mkdir -p /data/arbiter/
```

## 3.db01创建配置文件

#### master节点配置文件

```
cat >/opt/master/conf/mongod.conf<<EOF   
systemLog:
  destination: file 
  logAppend: true 
  path: /opt/master/log/mongodb.log

storage:
  journal:
    enabled: true
  dbPath: /data/master/
  directoryPerDB: true

  wiredTiger:
    engineConfig:
      cacheSizeGB: 1
      directoryForIndexes: true
    collectionConfig:
      blockCompressor: zlib
    indexConfig:
      prefixCompression: true

processManagement:
  fork: true
  pidFilePath: /opt/master/pid/mongodb.pid
  timeZoneInfo: /usr/share/zoneinfo

net:
  port: 28100
  bindIp: 127.0.0.1,$(ifconfig eth0|awk 'NR==2{print $2}')

replication:
  oplogSizeMB: 1024 
  replSetName: shard1

sharding:
  clusterRole: shardsvr
EOF
```

#### slave节点配置文件

```
cat >/opt/slave/conf/mongod.conf<<EOF   
systemLog:
  destination: file 
  logAppend: true 
  path: /opt/slave/log/mongodb.log

storage:
  journal:
    enabled: true
  dbPath: /data/slave/
  directoryPerDB: true

  wiredTiger:
    engineConfig:
      cacheSizeGB: 1
      directoryForIndexes: true
    collectionConfig:
      blockCompressor: zlib
    indexConfig:
      prefixCompression: true

processManagement:
  fork: true
  pidFilePath: /opt/slave/pid/mongodb.pid
  timeZoneInfo: /usr/share/zoneinfo

net:
  port: 28200
  bindIp: 127.0.0.1,$(ifconfig eth0|awk 'NR==2{print $2}')

replication:
  oplogSizeMB: 1024 
  replSetName: shard2

sharding:
  clusterRole: shardsvr
EOF
```

#### aribiter节点配置文件

```
cat >/opt/arbiter/conf/mongod.conf<<EOF   
systemLog:
  destination: file 
  logAppend: true 
  path: /opt/arbiter/log/mongodb.log

storage:
  journal:
    enabled: true
  dbPath: /data/arbiter/
  directoryPerDB: true

  wiredTiger:
    engineConfig:
      cacheSizeGB: 1
      directoryForIndexes: true
    collectionConfig:
      blockCompressor: zlib
    indexConfig:
      prefixCompression: true

processManagement:
  fork: true
  pidFilePath: /opt/arbiter/pid/mongodb.pid
  timeZoneInfo: /usr/share/zoneinfo

net:
  port: 28300
  bindIp: 127.0.0.1,$(ifconfig eth0|awk 'NR==2{print $2}')

replication:
  oplogSizeMB: 1024 
  replSetName: shard3

sharding:
  clusterRole: shardsvr
EOF
```

## 4.db02创建配置文件

#### master节点配置文件

```
cat >/opt/master/conf/mongod.conf<<EOF   
systemLog:
  destination: file 
  logAppend: true 
  path: /opt/master/log/mongodb.log

storage:
  journal:
    enabled: true
  dbPath: /data/master/
  directoryPerDB: true

  wiredTiger:
    engineConfig:
      cacheSizeGB: 1
      directoryForIndexes: true
    collectionConfig:
      blockCompressor: zlib
    indexConfig:
      prefixCompression: true

processManagement:
  fork: true
  pidFilePath: /opt/master/pid/mongodb.pid
  timeZoneInfo: /usr/share/zoneinfo

net:
  port: 28100
  bindIp: 127.0.0.1,$(ifconfig eth0|awk 'NR==2{print $2}')

replication:
  oplogSizeMB: 1024 
  replSetName: shard2

sharding:
  clusterRole: shardsvr
EOF
```

#### slave节点配置文件

```
cat >/opt/slave/conf/mongod.conf<<EOF   
systemLog:
  destination: file 
  logAppend: true 
  path: /opt/slave/log/mongodb.log

storage:
  journal:
    enabled: true
  dbPath: /data/slave/
  directoryPerDB: true

  wiredTiger:
    engineConfig:
      cacheSizeGB: 1
      directoryForIndexes: true
    collectionConfig:
      blockCompressor: zlib
    indexConfig:
      prefixCompression: true

processManagement:
  fork: true
  pidFilePath: /opt/slave/pid/mongodb.pid
  timeZoneInfo: /usr/share/zoneinfo

net:
  port: 28200
  bindIp: 127.0.0.1,$(ifconfig eth0|awk 'NR==2{print $2}')

replication:
  oplogSizeMB: 1024 
  replSetName: shard3

sharding:
  clusterRole: shardsvr
EOF
```

#### aribiter节点配置文件

```
cat >/opt/arbiter/conf/mongod.conf<<EOF   
systemLog:
  destination: file 
  logAppend: true 
  path: /opt/arbiter/log/mongodb.log

storage:
  journal:
    enabled: true
  dbPath: /data/arbiter/
  directoryPerDB: true

  wiredTiger:
    engineConfig:
      cacheSizeGB: 1
      directoryForIndexes: true
    collectionConfig:
      blockCompressor: zlib
    indexConfig:
      prefixCompression: true

processManagement:
  fork: true
  pidFilePath: /opt/arbiter/pid/mongodb.pid
  timeZoneInfo: /usr/share/zoneinfo

net:
  port: 28300
  bindIp: 127.0.0.1,$(ifconfig eth0|awk 'NR==2{print $2}')

replication:
  oplogSizeMB: 1024 
  replSetName: shard1

sharding:
  clusterRole: shardsvr
EOF
```

## 5.db03创建配置文件

#### master节点配置文件

```
cat >/opt/master/conf/mongod.conf<<EOF   
systemLog:
  destination: file 
  logAppend: true 
  path: /opt/master/log/mongod.log

storage:
  journal:
    enabled: true
  dbPath: /data/master/
  directoryPerDB: true

  wiredTiger:
    engineConfig:
      cacheSizeGB: 1
      directoryForIndexes: true
    collectionConfig:
      blockCompressor: zlib
    indexConfig:
      prefixCompression: true

processManagement:
  fork: true
  pidFilePath: /opt/master/pid/mongodb.pid
  timeZoneInfo: /usr/share/zoneinfo

net:
  port: 28100
  bindIp: 127.0.0.1,$(ifconfig eth0|awk 'NR==2{print $2}')

replication:
  oplogSizeMB: 1024 
  replSetName: shard3

sharding:
  clusterRole: shardsvr
EOF
```

#### slave节点配置文件

```
cat >/opt/slave/conf/mongod.conf<<EOF   
systemLog:
  destination: file 
  logAppend: true 
  path: /opt/slave/log/mongodb.log

storage:
  journal:
    enabled: true
  dbPath: /data/slave/
  directoryPerDB: true

  wiredTiger:
    engineConfig:
      cacheSizeGB: 1
      directoryForIndexes: true
    collectionConfig:
      blockCompressor: zlib
    indexConfig:
      prefixCompression: true

processManagement:
  fork: true
  pidFilePath: /opt/slave/pid/mongod.pid
  timeZoneInfo: /usr/share/zoneinfo

net:
  port: 28200
  bindIp: 127.0.0.1,$(ifconfig eth0|awk 'NR==2{print $2}')

replication:
  oplogSizeMB: 1024 
  replSetName: shard1

sharding:
  clusterRole: shardsvr
EOF
```

#### aribiter节点配置文件

```
cat >/opt/arbiter/conf/mongod.conf<<EOF   
systemLog:
  destination: file 
  logAppend: true 
  path: /opt/arbiter/log/mongodb.log

storage:
  journal:
    enabled: true
  dbPath: /data/arbiter/
  directoryPerDB: true

  wiredTiger:
    engineConfig:
      cacheSizeGB: 1
      directoryForIndexes: true
    collectionConfig:
      blockCompressor: zlib
    indexConfig:
      prefixCompression: true

processManagement:
  fork: true
  pidFilePath: /opt/arbiter/pid/mongod.pid
  timeZoneInfo: /usr/share/zoneinfo

net:
  port: 28300
  bindIp: 127.0.0.1,$(ifconfig eth0|awk 'NR==2{print $2}')

replication:
  oplogSizeMB: 1024 
  replSetName: shard2

sharding:
  clusterRole: shardsvr
EOF
```

## 6.优化警告

注意！三台服务器都操作！！！

```
useradd mongod -s /sbin/nologin -M 
echo "never"  > /sys/kernel/mm/transparent_hugepage/enabled
echo "never"  > /sys/kernel/mm/transparent_hugepage/defrag
```

## 7.启动服务

```
mongod -f /opt/master/conf/mongod.conf 
mongod -f /opt/slave/conf/mongod.conf 
mongod -f /opt/arbiter/conf/mongod.conf
netstat -lntup|grep mongod
```

## 8.初始化副本集

### db01 master节点初始化shard1的副本

```
mongo --port 28100
rs.initiate()
rs.add("10.0.0.53:28200")
rs.addArb("10.0.0.52:28300")
```

### db02 master节点初始化shard2的副本

```
mongo --port 28100
config = {
    _id:"shard2", 
    members:[
        {_id:0,host:"10.0.0.52:28100"},
        {_id:1,host:"10.0.0.51:28200"},
        {_id:2,host:"10.0.0.53:28300",arbiterOnly:true},
    ] 
}
rs.initiate(config)
```

### db03 master节点初始化shard3的副本

```
mongo --port 28100
config = {
    _id:"shard3", 
    members:[
        {_id:0,host:"10.0.0.53:28100"},
        {_id:1,host:"10.0.0.52:28200"},
        {_id:2,host:"10.0.0.51:28300",arbiterOnly:true},
    ] 
}
rs.initiate(config)
```

## 9.检查命令

```
mongo --port 28100
rs.status()

mongod -f /opt/arbiter/conf/mongod.conf  --shutdown
mongod -f /opt/slave/conf/mongod.conf    --shutdown
mongod -f /opt/master/conf/mongod.conf   --shutdown
```

# 第6章 分片集群搭建config步骤

## 1.创建目录

```
mkdir -p /opt/config/{conf,log,pid}
mkdir -p /data/config/
```

## 2.创建配置文件

```
cat >/opt/config/conf/mongod.conf<<EOF   
systemLog:
  destination: file 
  logAppend: true 
  path: /opt/config/log/mongodb.log

storage:
  journal:
    enabled: true
  dbPath: /data/config/
  directoryPerDB: true

  wiredTiger:
    engineConfig:
      cacheSizeGB: 1
      directoryForIndexes: true
    collectionConfig:
      blockCompressor: zlib
    indexConfig:
      prefixCompression: true

processManagement:
  fork: true
  pidFilePath: /opt/config/pid/mongod.pid
  timeZoneInfo: /usr/share/zoneinfo

net:
  port: 40000
  bindIp: 127.0.0.1,$(ifconfig eth0|awk 'NR==2{print $2}')

replication:
  replSetName: configset

sharding:
  clusterRole: configsvr
EOF
```

## 3.启动

```
mongod -f /opt/config/conf/mongod.conf
```

## 4.初始化副本集

```
mongo localhost:40000
rs.initiate({
    _id:"configset", 
    configsvr: true,
    members:[
        {_id:0,host:"10.0.0.51:40000"},
        {_id:1,host:"10.0.0.52:40000"},
        {_id:2,host:"10.0.0.53:40000"},
    ] })
```

## 5.检查

```
rs.status()
```

# 第7章 分片集群搭建mongos配置

## 1.创建目录

```
mkdir -p /opt/mongos/{conf,log,pid}
```

## 2.创建配置文件

```
cat >/opt/mongos/conf/mongos.conf  <<EOF   
systemLog:
  destination: file 
  logAppend: true 
  path: /opt/mongos/log/mongos.log

processManagement:
  fork: true
  pidFilePath: /opt/mongos/pid/mongos.pid
  timeZoneInfo: /usr/share/zoneinfo

net:
  port: 60000
  bindIp: 127.0.0.1,$(ifconfig eth0|awk 'NR==2{print $2}')

sharding:
  configDB: 
    configset/10.0.0.51:40000,10.0.0.52:40000,10.0.0.53:40000
EOF
```

## 3.启动

```
mongos -f /opt/mongos/conf/mongos.conf
```

## 4.登录mongos

```
mongo --port 60000
```

## 5.添加分片成员

```
use admin
db.runCommand({addShard:'shard1/10.0.0.51:28100,10.0.0.53:28200,10.0.0.52:28300'})
db.runCommand({addShard:'shard2/10.0.0.52:28100,10.0.0.51:28200,10.0.0.53:28300'})
db.runCommand({addShard:'shard3/10.0.0.53:28100,10.0.0.52:28200,10.0.0.51:28300'})
```

## 6.查看分片信息

```
db.runCommand( { listshards : 1 } )
```

# 第8章 hash分片配置

## 1.数据库开启分片

```
mongo --port 60000
use admin
db.runCommand( { enablesharding : "oldboy" } )
```

## 2.创建索引

```
use oldboy
db.hash.ensureIndex( { id: "hashed" } )
```

## 3.集合开启哈希分片

```
use admin
sh.shardCollection( "oldboy.hash", { id: "hashed" } )
```

## 4.生成测试数据

```
use oldboy
for(i=1;i<10000;i++){ db.hash.insert({"id":i,"name":"shenzheng","age":70,"date":new Date()}); }
```

## 5.分片验证

```
shard1
mongo db01:28100
use oldboy
db.hash.count()
33755


shard2
mongo db02:28100
use oldboy
db.hash.count()
33142


shard3
mongo db03:28100
use oldboy
db.hash.count()
33102
```

# 第9章 分片集群常用管理命令

## 1.列出分片所有详细信息

```
db.printShardingStatus()
sh.status()
```

## 2.列出所有分片成员信息

```
use admin
db.runCommand({ listshards : 1})
```

## 3.列出开启分片的数据库

```
use config
db.databases.find({"partitioned": true })
```

## 4.查看分片的片键

```
use config
db.collections.find().pretty()
```

## 5.查看集合的分片信息

```
db.getCollection('range').getShardDistribution()
db.getCollection('hash').getShardDistribution()
```

# 第10章 示意图

![img](https://tva1.sinaimg.cn/large/00831rSTgy1gcjlwrm0xcj30u00uraf1.jpg)