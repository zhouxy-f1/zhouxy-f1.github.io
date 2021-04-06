# 第一章 关系型与非关系型

```
NoSQL  not only sql 
NoSQL，指的是非关系型的数据库。
NoSQL有时也称作Not Only SQL的缩写是对不同于传统的关系型数据库的数据库管理系统的统称。 
对NoSQL最普遍的解释是”非关联型的”，强调Key-Value Stores和文档数据库的优点，而不是单纯的RDBMS。 
NoSQL用于超大规模数据的存储。
这些类型的数据存储不需要固定的模式，无需多余操作就可以横向扩展。
今天我们可以通过第三方平台可以很容易的访问和抓取数据。
用户的个人信息，社交网络，地理位置，用户生成的数据和用户操作日志已经成倍的增加。
我们如果要对这些用户数据进行挖掘，那SQL数据库已经不适合这些应用了
NoSQL数据库的发展也却能很好的处理这些大的数据。
```

# 第二章 mongo和mysql数据对比

```
mysql   mongo 
库       库
表       集合
字段  key:value
行       文档 

name        age  job  
oldzhang    28   it 
xiaozhang   28   it
xiaofei     18   student  SZ 

{name:'oldzhang',age:'28',job:'it'},
{name:'xiaozhang',age:'28',job:'it'},
{name:'xiaozhang',age:'28',job:'it',host:'SZ'}
```

# 第三章 MongoDB特点

```
高性能： 
Mongodb提供高性能的数据持久性
尤其是支持嵌入式数据模型减少数据库系统上的I/O操作
索引支持能快的查询，并且可以包括来嵌入式文档和数组中的键

丰富的语言查询：
Mongodb支持丰富的查询语言来支持读写操作（CRUD）以及数据汇总，文本搜索和地理空间索引 

高可用性： 
Mongodb的复制工具，成为副本集，提供自动故障转移和数据冗余

水平可扩展性：
Mongodb提供了可扩展性，作为其核心功能的一部分，分片是将数据分，在一组计算机上

支持多种存储引擎： 
WiredTiger存储引擎和、MMAPv1存储引擎和InMemory存储引擎
```

# 第四章 mongo应用场景

```
游戏场景，使用 MongoDB 存储游戏用户信息，用户的装备、积分等直接以内嵌文档的形式存储，方便查询、更新

物流场景，使用 MongoDB 存储订单信息，订单状态在运送过程中会不断更新，以 MongoDB 内嵌数组的形式来存储，一次查询就能将订单所有的变更读取出来。

社交场景，使用 MongoDB 存储存储用户信息，以及用户发表的朋友圈信息，通过地理位置索引实现附近的人、地点等功能

物联网场景，使用 MongoDB 存储所有接入的智能设备信息，以及设备汇报的日志信息，并对这些信息进行多维度的分析

视频直播，使用 MongoDB 存储用户信息、礼物信息等,用户评论

电商场景，使用 MongoDB
商城上衣和裤子两种商品，除了有共同属性，如产地、价格、材质、颜色等外，还有各自有不同的属性集，如上衣的独有属性是肩宽、胸围、袖长等，裤子的独有属性是臀围、脚口和裤长等
```

# 第五章 安装部署mongodb

## 1.规划目录

```
#软件所在目录
/opt/mongodb
#单节点目录
/opt/mongo_27017/{conf,log,pid}
#数据目录
/data/mongo_27017
```

## 2.下载并解压

```
yum install libcurl openssl -y
上传软件
tar zxf mongodb-linux-x86_64-3.6.13.tgz -C /opt/
cd /opt/
ln -s mongodb-linux-x86_64-3.6.13 mongodb
```

## 3.创建文件目录以及数据目录

```
mkdir -p /opt/mongo_27017/{conf,log,pid}
mv /data /data_bak
mkdir -p /data/mongo_27017 
```

## 4.创建配置文件

```
cat >/opt/mongo_27017/conf/mongodb.conf<<EOF
systemLog:
 destination: file   
 logAppend: true  
 path: /opt/mongo_27017/log/mongodb.log

storage:
 journal:
   enabled: true
 dbPath: /data/mongo_27017
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
 pidFilePath: /opt/mongo_27017/pid/mongod.pid

net:
 port: 27017
 bindIp: 127.0.0.1,10.0.0.51
EOF
```

## 5.启动mongo

```
/opt/mongodb/bin/mongod -f /opt/mongo_27017/conf/mongodb.conf
```

## 6.检查是否启动

```
netstat -lntup|grep 27017
```

# 第七章 配置登录mongo

## 1.写入环境变量

```
vim /etc/profile
export PATH=/opt/mongodb/bin:$PATH
source /etc/profile
```

## 2.登录

```
mongo
```

## 3.关闭

```
方法1：
使用localhost登录
mongo localhost:27017
use admin
db.shutdownServer()

方法2：
mongod -f /opt/mongo_27017/conf/mongodb.conf --shutdown
```

# 第八章 优化告警

## 1.访问控制

```
WARNING: Access control is not enabled for the database.
Read and write access to data and configuration is unrestricted.
```

解决方法：

```
开启安全认证功能
```

## 2.以root用户运行

```
WARNING: You are running this process as the root user, which is not recommended.
```

解决步骤：

```
mongod -f /opt/mongo_27017/conf/mongodb.conf --shutdown
useradd mongo
echo '123456'|passwd --stdin mongo
chown -R mongo:mongo /opt/
chown -R mongo:mongo /data/
su - mongo
mongod -f /opt/mongo_27017/conf/mongodb.conf
mongo
```

## 3.关闭大内存页技术

```
WARNING: /sys/kernel/mm/transparent_hugepage/enabled is 'always'.
We suggest setting it to 'never'
WARNING: /sys/kernel/mm/transparent_hugepage/defrag is 'always'.
We suggest setting it to 'never'
```

解决方法：
临时解决：

```
echo "never" >/sys/kernel/mm/transparent_hugepage/enabled
echo "never" >/sys/kernel/mm/transparent_hugepage/defrag
```

写入开机自启动：

```
chmod +x /etc/rc.d/rc.local
vim /etc/rc.d/rc.local
echo "never" >/sys/kernel/mm/transparent_hugepage/enabled
echo "never" >/sys/kernel/mm/transparent_hugepage/defrag
```

验证：

```
mongod -f /opt/mongo_27017/conf/mongodb.conf --shutdown
mongod -f /opt/mongo_27017/conf/mongodb.conf 
mongo 
```

## 4.解决rlimits太低

```
WARNING: soft rlimits too low. rlimits set to 31771 processes, 65535 files. Number of processes should be at least 32767.5 : 0.5 times number of files.
```

解决方法：

```
vim /etc/profile
ulimit -f unlimited
ulimit -t unlimited
ulimit -v unlimited
ulimit -n 64000
ulimit -m unlimited
ulimit -u 64000
```

生效配置:

```
source /etc/profile
```

验证：

```
mongod -f /opt/mongo_27017/conf/mongodb.conf --shutdown
mongod -f /opt/mongo_27017/conf/mongodb.conf 
mongo 
```

# 第九章 mongo基础命令

## 1.默认数据库说明

```
test:登录时默认存在的库
admin库:系统预留库,MongoDB系统管理库
local库:本地预留库,存储关键日志
config库:MongoDB配置信息库
```

## 2.查看数据库命令

```
show databases/show dbs
show tables/show collections
use admin
db/select database()
```

## 3.插入命令

### 3.1 插入单条

```
db.user_info.insert({"name":"zhangya","age":27,"ad":"北京市朝阳区"})
db.user_info.insert({"name":"zhangya","age":27,"ad":"北京市朝阳区"})
db.user_info.insert({"name":"yazhang","age":28,"ad":"北京市朝阳区"})
db.user_info.insert({"name":"xiaozhang","age":28,"ad":"北京市朝阳区"})
db.user_info.insert({"name":"xiaozhang","age":28,"ad":"北京市朝阳区","sex":"boy"})
```

### 3.2 插入多条

```
db.inventory.insertMany( [
    { "item": "journal", "qty": 25, "size": { "h": 14, "w": 21, "uom": "cm" }, "status": "A" },
    { "item": "notebook", "qty": 50, "size": { "h": 8.5, "w": 11, "uom": "in" }, "status": "A" },
    { "item": "paper", "qty": 100, "size": { "h": 8.5, "w": 11, "uom": "in" }, "status": "D" },
    { "item": "planner", "qty": 75, "size": { "h": 22.85, "w": 30, "uom": "cm" }, "status": "D" },
    { "item": "postcard", "qty": 45, "size": { "h": 10, "w": 15.25, "uom": "cm" }, "status": "A" }
]);
```

## 4.查询命令

### 4.1 查询一条

```
db.user_info.findOne()
```

### 4.2 查询所有

```
db.user_info.find()
```

### 4.3 查询符合条件

```
db.user_info.find({"age":28})
select * from user_info where age = 28;
```

### 4.4 查询嵌套的条件

```
db.inventory.find( { "size.uom": "in" } )
db.inventory.find( 
    { 
        "size.uom": "in" 
    } 
)
```

### 4.5 逻辑查询：and

```
db.inventory.find( { "size.uom": "cm" ,"status" : "A"} )
db.inventory.find( 
    { 
        "size.uom": "cm" ,
        "status" : "A"
    } 
)
```

### 4.6 逻辑查询 或

```
db.inventory.find(
    {
        $or:[
                {status:"D"},
                {qty:{$lt:30}}
            ]
    }
)
```

### 4.7 逻辑查询+或+and+正则表达式

```
db.inventory.find({status:"A",$or:[{qty:{$lt:30}},{item:/^p/}]})
db.inventory.find( 
    {
        status: "A",
        $or: [ 
                { qty: { $lt: 30 } }, 
                { item: /^p/ } 
             ]
    } 
)

db.inventory.find( 
    {
        status: "A",
        $or: [ 
                { qty: { $gt: 30 } }, 
                { item: /^p/ } 
             ]
    } 
)
```

## 5.更新数据

### 5.1 更改匹配条件的单条数据

```
db.inventory.find({ "item" : "paper" })
db.inventory.updateOne{ "item" : "paper" },{$set: {  "size.uom" : "cm",  "status" : "P" }})
db.inventory.updateOne(
    { "item" : "paper" },
    {
      $set: {  
                "size.uom" : "cm",  
                "status" : "P" 
            }
    }
)
```

### 5.2 更改匹配条件的多条数据

```
db.inventory.find({ "qty" : { $lt: 50 } })
db.inventory.updateMany(
    { "qty" : { $lt: 50 } },
    {
       $set: 
            { 
                "size.uom" : "mm", 
                "status": "P" 
            }
    }
)
```

### 5.3 添加字段

```
db.user_info.find({ "age" : 27})
db.user_info.updateMany(
    { "age" : 27},
    {
       $set: 
            { 
                "pet" : "cat"
            }
    }
)
```

## 6.索引

### 6.1 查看执行计划

```
db.user_info.find({"age":{ $lt: 30 }})
db.user_info.find({"age":{ $lt: 30 }}).explain()
```

### 6.2 创建索引

```
db.user_info.createIndex({ age: 1 },{background: true})
```

### 6.3 查看索引

```
db.user_info.getIndexes()
```

### 6.4 再次查看执行计划

```
db.user_info.find({"age":{ $lt: 30 }}).explain()
```

### 6.5 关键词

```
"stage" : "IXSCAN"
"indexName" : "age_1"
```

### 6.6 删除索引

```
db.user_info.dropIndex("age_1")
```

### 6.7 其他索引类型

```
COLLSCAN – Collection scan
IXSCAN – Scan of data in index keys
FETCH – Retrieving documents
SHARD_MERGE – Merging results from shards
SORT – Explicit sort rather than using index orde
```

## 7.删除

### 7.1 先查找需要删除的数据

```
db.inventory.find({"status":"P"})
```

### 7.2 删除单条

```
db.inventory.deleteOne({"status":"P"})
```

### 7.3 删除多个

```
db.inventory.deleteMany({"status":"P"})
```

### 7.4 删除索引

```
db.user_info.dropIndex("age_1")
```

### 7.5 删除集合

```
show dbs
db 
show tables
db.inventory.drop()
```

### 7.6 删除库

```
show dbs
db 
db.dropDatabase()
```

# 第十章 mongo工具

## 0.命令介绍

```
mongod          #启动命令
mongo           #登录命令         
mongodump       #备份导出，全备压缩
mongorestore    #恢复
mongoexport     #备份，数据可读json
mongoimport     #恢复
mongostat       #查看mongo运行状态
mongotop        #查看mongo运行状态
mongos          #集群分片命令
```

## 1.mongostat各字段解释说明：

```
insert/s : 官方解释是每秒插入数据库的对象数量，如果是slave，则数值前有*,则表示复制集操作
query/s : 每秒的查询操作次数
update/s : 每秒的更新操作次数
delete/s : 每秒的删除操作次数
getmore/s: 每秒查询cursor(游标)时的getmore操作数
command: 每秒执行的命令数，在主从系统中会显示两个值(例如 3|0),分表代表 本地|复制 命令
注： 一秒内执行的命令数比如批量插入，只认为是一条命令（所以意义应该不大）
dirty: 仅仅针对WiredTiger引擎，官网解释是脏数据字节的缓存百分比
used:仅仅针对WiredTiger引擎，官网解释是正在使用中的缓存百分比
flushes:
For WiredTiger引擎：指checkpoint的触发次数在一个轮询间隔期间
For MMAPv1 引擎：每秒执行fsync将数据写入硬盘的次数
注：一般都是0，间断性会是1， 通过计算两个1之间的间隔时间，可以大致了解多长时间flush一次。flush开销是很大的，如果频繁的flush，可能就要找找原因了
vsize： 虚拟内存使用量，单位MB （这是 在mongostat 最后一次调用的总数据）
res:  物理内存使用量，单位MB （这是 在mongostat 最后一次调用的总数据）
注：这个和你用top看到的一样, vsize一般不会有大的变动， res会慢慢的上升，如果res经常突然下降，去查查是否有别的程序狂吃内存。

qr: 客户端等待从MongoDB实例读数据的队列长度
qw：客户端等待从MongoDB实例写入数据的队列长度
ar: 执行读操作的活跃客户端数量
aw: 执行写操作的活客户端数量
注：如果这两个数值很大，那么就是DB被堵住了，DB的处理速度不及请求速度。看看是否有开销很大的慢查询。如果查询一切正常，确实是负载很大，就需要加机器了
netIn:MongoDB实例的网络进流量
netOut：MongoDB实例的网络出流量
注：此两项字段表名网络带宽压力，一般情况下，不会成为瓶颈
conn: 打开连接的总数，是qr,qw,ar,aw的总和
注：MongoDB为每一个连接创建一个线程，线程的创建与释放也会有开销，所以尽量要适当配置连接数的启动参数，maxIncomingConnections，阿里工程师建议在5000以下，基本满足多数场景
```

# 第十一章 创建用户和角色

## 0.与用户相关的命令

```
db.auth() 将用户验证到数据库。
db.changeUserPassword() 更改现有用户的密码。
db.createUser() 创建一个新用户。
db.dropUser() 删除单个用户。
db.dropAllUsers() 删除与数据库关联的所有用户。
db.getUser() 返回有关指定用户的信息。
db.getUsers() 返回有关与数据库关联的所有用户的信息。
db.grantRolesToUser() 授予用户角色及其特权。
db.removeUser() 已过时。从数据库中删除用户。
db.revokeRolesFromUser() 从用户中删除角色。
db.updateUser() 更新用户数据。
```

## 1.创建管理用户

```
mongo db01:27017
use admin 
db.createUser(
    {
        user: "admin",
        pwd: "123456",
        roles:[ 
                { 
                    role: "root", 
                    db:"admin"
                }
              ]
    }   
)
```

## 2.查看创建的用户

```
db.getUsers()
```

## 3.配置文件添加权限认证参数

```
security:     
  authorization: enabled
```

## 4.重启mongo

```
mongod -f /opt/mongo_27017/conf/mongodb.conf --shutdown
mongod -f /opt/mongo_27017/conf/mongodb.conf
```

## 5.使用admin用户登录

```
mongo db01:27017 -uadmin -p --authenticationDatabase admin
```

## 6.创建其他用户

```
use test
db.createUser(
  {
    user: "mysun",
    pwd: "123456",
    roles: [ { role: "readWrite", db: "write" },
             { role: "read", db: "read" } ]
  }
)
```

## 7.创建测试数据

```
use write
db.write.insert({"name":"zhangya","age":27,"ad":"北京市朝阳区"})
db.write.insert({"name":"zhangya","age":27,"ad":"北京市朝阳区"})
db.write.insert({"name":"yazhang","age":28,"ad":"北京市朝阳区"})
db.write.insert({"name":"xiaozhang","age":28,"ad":"北京市朝阳区"})
db.write.insert({"name":"xiaozhang","age":28,"ad":"北京市朝阳区","sex":"boy"})

use read
db.read.insert({"name":"zhangya","age":27,"ad":"北京市朝阳区"})
db.read.insert({"name":"zhangya","age":27,"ad":"北京市朝阳区"})
db.read.insert({"name":"yazhang","age":28,"ad":"北京市朝阳区"})
db.read.insert({"name":"xiaozhang","age":28,"ad":"北京市朝阳区"})
db.read.insert({"name":"xiaozhang","age":28,"ad":"北京市朝阳区","sex":"boy"})
```

## 8.退出admin，使用mysun用户登录

```
mongo db01:27017 -umysun -p --authenticationDatabase test
use write
db.write.find()
db.write.insert({"name":"zhangya","age":27,"ad":"北京市朝阳区"})

use read
db.read.find()
db.read.insert({"name":"zhangya","age":27,"ad":"北京市朝阳区"})
```

## 9.修改用户权限

```
use test
db.updateUser(
  'mysun',
  { 
    pwd: "123456",
    roles: [ { role: "readWrite", db: "write" },
             { role: "readWrite", db: "read" } ,
             { role: "readWrite", db: "test" }
             ]
             
  }
)
```

## 10.删除用户

```
db.getUsers()
db.dropUser('mysun')
```

# 第十二章 mongo副本集配置

## 1.创建节点目录和数据目录

```
su - mongo 
mongod -f /opt/mongo_27017/conf/mongodb.conf --shutdown
mkdir -p /opt/mongo_2801{7,8,9}/{conf,log,pid}  
mkdir -p /data/mongo_2801{7,8,9}
```

## 2.创建配置文件

```
cat >/opt/mongo_28017/conf/mongo_28017.conf <<EOF
systemLog:
  destination: file   
  logAppend: true  
  path: /opt/mongo_28017/log/mongodb.log

storage:
  journal:
    enabled: true
  dbPath: /data/mongo_28017
  directoryPerDB: true
  wiredTiger:
     engineConfig:
        cacheSizeGB: 0.5 
        directoryForIndexes: true
     collectionConfig:
        blockCompressor: zlib
     indexConfig:
        prefixCompression: true

processManagement:
  fork: true
  pidFilePath: /opt/mongo_28017/pid/mongod.pid

net:
  port: 28017
  bindIp: 127.0.0.1,10.0.0.51

replication:
   oplogSizeMB: 1024 
   replSetName: dba
EOF
```

## 3.复制配置文件到其他节点

```
cp /opt/mongo_28017/conf/mongo_28017.conf /opt/mongo_28018/conf/mongo_28018.conf
cp /opt/mongo_28017/conf/mongo_28017.conf /opt/mongo_28019/conf/mongo_28019.conf
```

## 4.替换端口号

```
sed -i 's#28017#28018#g' /opt/mongo_28018/conf/mongo_28018.conf  
sed -i 's#28017#28019#g' /opt/mongo_28019/conf/mongo_28019.conf
```

## 5.启动所有节点

```
mongod -f /opt/mongo_28017/conf/mongo_28017.conf
mongod -f /opt/mongo_28018/conf/mongo_28018.conf
mongod -f /opt/mongo_28019/conf/mongo_28019.conf
```

## 6.初始化集群

```
config = {
            _id : "dba", 
            members : [
                        {_id : 0, host : "db01:28017"},
                        {_id : 1, host : "db01:28018"},
                        {_id : 2, host : "db01:28019"},
            ]}
rs.initiate(config) 
```

## 7.插入数据

```
db.inventory.insertMany( [
    { "item": "journal", "qty": 25, "size": { "h": 14, "w": 21, "uom": "cm" }, "status": "A" },
    { "item": "notebook", "qty": 50, "size": { "h": 8.5, "w": 11, "uom": "in" }, "status": "A" },
    { "item": "paper", "qty": 100, "size": { "h": 8.5, "w": 11, "uom": "in" }, "status": "D" },
    { "item": "planner", "qty": 75, "size": { "h": 22.85, "w": 30, "uom": "cm" }, "status": "D" },
    { "item": "postcard", "qty": 45, "size": { "h": 10, "w": 15.25, "uom": "cm" }, "status": "A" }
]);
```

## 8.副本节点登录查看数据

```
rs.slaveOk()
use test
db.inventory.find()
```

## 9.设置副本可读

方法1:临时生效

```
rs.slaveOk()
```

方法2:写入启动文件

```
echo "rs.slaveOk()" > ~/.mongorc.js
```

# 第十三章 副本集权重调整

## 0.模拟故障转移

```
mongod -f /opt/mongo_28017/conf/mongo_28017.conf --shutdown
mongod -f /opt/mongo_28017/conf/mongo_28017.conf
```

## 1.查看当前副本集配置

```
rs.conf()
```

## 2.设置权重

```
config=rs.conf()
config.members[0].priority=100
rs.reconfig(config)
```

## 3.主节点主动降级

```
rs.stepDown()
```

## 4.恢复成默认的权重

```
config=rs.conf()
config.members[0].priority=1
rs.reconfig(config)
```

# 第十四章 增加新节点和删除旧节点

## 1.创建新节点并启动

```
mkdir -p /opt/mongo_28010/{conf,log,pid}
mkdir -p /data/mongo_28010
cp /opt/mongo_28017/conf/mongo_28017.conf /opt/mongo_28010/conf/mongo_28010.conf
sed -i 's#28017#28010#g' /opt/mongo_28010/conf/mongo_28010.conf
mongod -f /opt/mongo_28010/conf/mongo_28010.conf
mongo db01:28010
```

## 2.集群添加节点

```
mongo db01:28017
use admin
rs.add("db01:28010")
```

## 3.新节点查看信息

```
mongo db01:28010
```

## 4.删除节点

```
rs.remove("db01:28010")
rs.remove("db01:28011")
```

# 第十五 仲裁节点

## 1.创建新节点并启动

```
mkdir -p /opt/mongo_28011/{conf,log,pid}
mkdir -p /data/mongo_28011
cp /opt/mongo_28017/conf/mongo_28017.conf /opt/mongo_28011/conf/mongo_28011.conf
sed -i 's#28017#28011#g' /opt/mongo_28011/conf/mongo_28011.conf
mongod -f /opt/mongo_28011/conf/mongo_28011.conf
mongo db01:28011
```

## 2.将仲裁节点加入集群

```
rs.addArb("db01:28010")
```

# 第十六章 mongo备份与恢复

## 1.工具介绍

```
（1）mongoexport/mongoimport
（2）mongodump/mongorestore
```

## 2.应用场景

```
1.异构平台迁移  mysql <---> mongodb
2.同平台，跨大版本：mongodb 2  ----> mongodb 3
mongoexport/mongoimport:json csv
```

日常备份恢复时使用.

```
mongodump/mongorestore
```

## 3.导出工具mongoexport

单表备份

```
mongoexport --port 27017 -d test -c inventory -o /data/inventory.json
```

单表备份至csv格式

```
mongoexport --port 27017 -d test -c user_info --type=csv -f name,age,ad -o /data/user_info.csv
```

## 4.恢复

```
mongoimport --port 27017 -d test -c inventory /data/inventory.json
mongoimport --port 27017 -d test -c user_info --type=csv --headerline --file  /data/user_info.csv
```

## 5.mysql数据迁移到mongo

```
select * from world.city into outfile '/tmp/city.csv' fields terminated by ',';
```

编辑csv文件,添加列名

```
ID,Name,CountryCode,District,Population
mongoimport --port 27017 -d world -c city --type=csv --headerline --file  /data/city.csv
mongoexport --port 27017 -d world -c city -o /data/city.json
```

## 6.导出与恢复

```
mongodump  --port 27017 -o /data/backup
mongorestore --port 27017 -d world  /data/backup/world/ --drop
mongorestore --port 27017 /data/backup/ --drop
```

# 第十七章 模拟误删除恢复数据

## 1.准备测试数据

```
use backup 
db.backup.insertMany( [
    { "id": 1},
    { "id": 2},
    { "id": 3},
]);
```

## 2.全备环境

```
rm -rf /data/backup/*
mongodump --port 28017 --oplog -o /data/backup
```

## 3.增加新数据

```
mongo db01:28017
use backup 
db.backup.insertMany( [
    { "id": 4},
    { "id": 5},
    { "id": 6},
]);
```

## 4.模拟删除集合

```
mongo db01:28017
use backup 
db.backup.drop()
```

## 5.备份oplog

```
mongodump --port 28017 -d local -c oplog.rs  -o /data/backup
```

## 6.查找误操作时间点

```
use local 
db.oplog.rs.find({ns:"backup.$cmd"}).pretty();
```

## 7.找到时间点信息

```
"ts" : Timestamp(1575023546, 1),
```

## 8.恢复数据

```
cd /data/backup/local/
cp oplog.rs.bson ../oplog.bson
rm -rf /data/backup/local/
mongorestore --port 28017 --oplogReplay --oplogLimit "1575023546:1"  --drop  /data/backup/
```