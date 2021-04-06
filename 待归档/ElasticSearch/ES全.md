# 第1章 Elasticsearch介绍

## 1.什么是全文检索和Lucene

```
基于java环境,基于Lucene之上包装一层外壳
Lucene是一个java的搜索引擎库,操作非常繁琐

全文检索和倒排索引:
数据里里的标题:
1.老男孩教育             1
2.老男孩教育linux学院      1   1   1
3.老男孩教育python学院     1   1 
4.老男孩教育DBA              1   1
5.老男孩教育oldzhang     1

ES内部分词,评分,倒排索引:
老男孩     1 2 3 4 5   
教育  1 2 3 4 5 
学院    2 3 
linux   2
python  3
DBA     4

用户输入:
老男孩学院
linux老男孩学院DBA
```

## 2.Elasticsearch应用场景

```
1.搜索: 电商,百科,app搜索
2.高亮显示: github 
3.分析和数据挖掘: ELK
```

## 3.Elasticsearch特点

```
1.高性能,天然分布式
2.对运维友好,不需要会java语言,开箱即用
3.功能丰富
```

## 4.Elasticsearch在电商搜索的实现

```
mysql:
skuid   name  
1       狗粮100kg
2       猫粮50kg
3       猫罐头200g

ES:
聚合运算之后得到SKUID:
1
2

拿到ID之后,mysql就只需要简单地where查询即可
mysql:
select xx from xxx where skuid 1 
```

# 第2章 ES安装启动

## 0.关闭防火墙

```
iptables -nL
iptables -F
iptables -X
iptables -Z
iptables -nL
```

## 1.下载软件

```
mkdir /data/soft
[root@db-01 /data/soft]# ll -h
total 268M
-rw-r--r-- 1 root root 109M Feb 25  2019 elasticsearch-6.6.0.rpm
-rw-r--r-- 1 root root 159M Sep  2 16:35 jdk-8u102-linux-x64.rpm
```

## 2.安装jdk

```
rpm -ivh jdk-8u102-linux-x64.rpm 
[root@db-01 /data/soft]# java -version
openjdk version "1.8.0_212"
OpenJDK Runtime Environment (build 1.8.0_212-b04)
OpenJDK 64-Bit Server VM (build 25.212-b04, mixed mode)
```

## 3.安装ES

```
rpm -ivh elasticsearch-6.6.0.rpm
```

## 4.启动并检查

```
systemctl daemon-reload
systemctl enable elasticsearch.service
systemctl start elasticsearch.service

netstat -lntup|grep 9200

[root@db01 /data/soft]# curl 127.0.0.1:9200
{
  "name" : "pRG0qLR",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "mNuJSe07QM61IOxecnanZg",
  "version" : {
    "number" : "6.6.0",
    "build_flavor" : "default",
    "build_type" : "rpm",
    "build_hash" : "a9861f4",
    "build_date" : "2019-01-24T11:27:09.439740Z",
    "build_snapshot" : false,
    "lucene_version" : "7.6.0",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

# 第3章 ES自定义配置

## 1.查看ES有哪些配置

```
[root@db01 ~]# rpm -qc elasticsearch 
/etc/elasticsearch/elasticsearch.yml        #ES的主配置文件
/etc/elasticsearch/jvm.options              #jvm虚拟机配置
/etc/sysconfig/elasticsearch                #默认一些系统配置参数
/usr/lib/sysctl.d/elasticsearch.conf        #配置参数,不需要改动
/usr/lib/systemd/system/elasticsearch.service   #system启动文件
```

## 2.自定义配置文件

```
cp /etc/elasticsearch/elasticsearch.yml  /opt/
cat >/etc/elasticsearch/elasticsearch.yml<<EOF
node.name: node-1
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
bootstrap.memory_lock: true
network.host: 10.0.0.51,127.0.0.1
http.port: 9200
EOF
```

## 3.重启服务

```
systemctl restart elasticsearch.service
```

## 4.解决内存锁定失败

重启后查看日志发现提示内存锁定失败

```
[root@db01 ~]# tail -f /var/log/elasticsearch/elasticsearch.log 
[2019-11-14T09:42:29,513][ERROR][o.e.b.Bootstrap          ] [node-1] node validation exception
[1] bootstrap checks failed
[1]: memory locking requested for elasticsearch process but memory is not locked
```

解决方案:

```
systemctl edit elasticsearch
[Service]
LimitMEMLOCK=infinity

systemctl daemon-reload
systemctl restart elasticsearch.service
```

# 第4章 es-head插件安装

注意:需要修改配置文件添加允许跨域参数

```
http.cors.enabled: true 
http.cors.allow-origin: "*"
```

## 1.es-head 三种方式

1.npm安装方式

2.docker安装

3.google浏览器插件（推荐）

```
从google商店安装es-head插件
将安装好的插件导出到本地
修改插件文件名为zip后缀
解压目录
拓展程序-开发者模式-打开已解压的目录
连接地址修改为ES的IP地址
```

## 2.具体操作命令

Head插件在5.0以后安装方式发生了改变，需要nodejs环境支持，或者直接使用别人封装好的docker镜像

插件官方地址
https://github.com/mobz/elasticsearch-head

使用docker部署elasticsearch-head

```
docker pull alivv/elasticsearch-head
docker run --name es-head -p 9100:9100 -dit elivv/elasticsearch-head
```

使用nodejs编译安装elasticsearch-head

```
cd /opt/
wget https://nodejs.org/dist/v12.13.0/node-v12.13.0-linux-x64.tar.xz
tar xf node-v12.13.0-linux-x64.tar.xz
mv node-v12.13.0-linux-x64 node
echo "PATH=\$PATH:/opt/node/bin" >> /etc/profile
source profile 
npm -v
node -v 
git clone git://github.com/mobz/elasticsearch-head.git
unzip elasticsearch-head-master.zip
cd elasticsearch-head-master
npm install -g cnpm --registry=https://registry.npm.taobao.org
cnpm install
npm run start &
```

# 第5章 kibana与ES交互

## 1.安装kibana

```
rpm -ivh kibana-6.6.0-x86_64.rpm
```

## 2.配置kibana

```
[root@db-01 /data/soft]# grep "^[a-Z]" /etc/kibana/kibana.yml 
server.port: 5601
server.host: "10.0.0.51"
elasticsearch.hosts: ["http://localhost:9200"]
kibana.index: ".kibana"
```

## 3.启动kibana

```
systemctl start kibana
```

# 第6章 插入命令

## 1.使用自定义的ID

```
PUT oldzhang/info/1
{
  "name": "zhang",
  "age": "29"
}
```

## 2.使用随机ID

```
POST oldzhang/info/
{
  "name": "zhang",
  "age": "29",
  "pet": "xiaoqi"
}
```

## 3.和mysql对应关系建议单独列一个id字段

```
POST oldzhang/info/
{
  "uid": "1",
  "name": "ya",
  "age": "29"
}
```

# 第7章 查询命令

## 1.创建测试语句

```
POST oldzhang/info/
{
  "name": "zhang",
  "age": "29",
  "pet": "xiaoqi",
  "job": "it"
}

POST oldzhang/info/
{
  "name": "xiao1",
  "age": "30",
  "pet": "xiaoqi",
  "job": "it"
}

POST oldzhang/info/
{
  "name": "xiao2",
  "age": "26",
  "pet": "xiaoqi",
  "job": "it"
}

POST oldzhang/info/
{
  "name": "xiao4",
  "age": "35",
  "pet": "xiaoqi",
  "job": "it"
}

POST oldzhang/info/
{
  "name": "ya",
  "age": "28",
  "pet": "xiaomin",
  "job": "it"
}

POST oldzhang/info/
{
  "name": "xiaomin",
  "age": "26",
  "pet": "xiaowang",
  "job": "SM"

}

POST oldzhang/info/
{
  "name": "hemengfei",
  "age": "38",
  "pet": "xiaohe",
  "job": "3P"
}

POST oldzhang/info/
{
  "name": "xiaoyu",
  "age": "28",
  "pet": "bijiben",
  "job": "fly"
}
```

## 2.简单查询

```
GET oldzhang/_search/
```

## 3.条件查询

```
GET oldzhang/_search
{
  "query": {
    "term": {
      "name": {
        "value": "xiaomin"
      }
    }
  }
}

GET oldzhang/_search
{
  "query": {
    "term": {
      "job": {
        "value": "it"
      }
    }
  }
}
```

## 4.多条件查询

```
GET /oldzhang/_search
{
    "query" : {
      "bool": {
        "must": [
          {"match": {"pet": "xiaoqi"}},
          {"match": {"name": "zhang"}}
        ],
        "filter": {
          "range": {
            "age": {
              "gte": 27,
              "lte": 30
            }
          }
          }
        }
      }
    }
}
```

# 第8章 集群相关名词

## 0.默认分片和副本规则

```
5分片
1副本
```

## 1.集群健康状态

```
绿色: 所有数据都完整，且副本数满足
黄色: 所有数据都完整，但是副本数不满足
红色: 一个或多个索引数据不完整
```

## 2.节点类型

```
主节点:     负责调度数据分配到哪个节点
数据节点:  实际负责处理数据的节点
默认:         主节点也是工作节点
```

## 3.数据分片

```
主分片:        实际存储的数据,负责读写,粗框的是主分片
副本分片:    主分片的副本,提供读,同步主分片,细框的是副本分片
```

## 4.副本

```
主分片的备份,副本数量可以自定义
```

# 第9章: 部署ES集群

## 1.安装java

```
rpm -ivh jdk-8u102-linux-x64.rpm
```

## 2.安装ES

```
rpm -ivh elasticsearch-6.6.0.rpm
```

## 3.配置内存锁定

```
systemctl edit elasticsearch.service
[Service]
LimitMEMLOCK=infinity
```

## 4.集群配置文件

### b01配置文件:

```
cat > /etc/elasticsearch/elasticsearch.yml <<EOF
cluster.name: linux5
node.name: node-1
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
bootstrap.memory_lock: true
network.host: 10.0.0.51,127.0.0.1
http.port: 9200
discovery.zen.ping.unicast.hosts: ["10.0.0.51", "10.0.0.52"]
discovery.zen.minimum_master_nodes: 1
EOF
```

### db02配置文件:

```
cat> /etc/elasticsearch/elasticsearch.yml <<EOF
cluster.name: linux5
node.name: node-2
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
bootstrap.memory_lock: true
network.host: 10.0.0.52,127.0.0.1
http.port: 9200
discovery.zen.ping.unicast.hosts: ["10.0.0.51", "10.0.0.52"]
discovery.zen.minimum_master_nodes: 1
EOF 
```

## 5.启动

```
systemctl daemon-reload
systemctl restart elasticsearch
```

## 6.查看日志

```
tail -f /var/log/elasticsearch/linux5.log
```

## 7.检查集群

```
ES-head查看是否有2个节点
```

# 第10章 集群注意事项

```
1.插入和读取数据在任意节点都可以执行,效果一样
2.es-head可以连接集群内任一台服务

3.主节点负责读写
如果主分片所在的节点坏掉了,副本分片会升为主分片

4.主节点负责调度
如果主节点坏掉了,数据节点会自动升为主节点

5.通讯端口
默认会有2个通讯端口：9200和9300
9300并没有在配置文件里配置过
如果开启了防火墙并且没有放开9300端口，那么集群通讯就会失败
```

# 第11章 查看集群各种信息

```
GET _cat/nodes
GET _cat/health
GET _cat/master
GET _cat/fielddata
GET _cat/indices
GET _cat/shards
GET _cat/shards/oldzhang
```

# 第12章 扩容第三台机器

## 1.安装java

```
rpm -ivh jdk-8u102-linux-x64.rpm
```

## 2.安装ES

```
rpm -ivh elasticsearch-6.6.0.rpm
```

## 3.配置内存锁定

```
systemctl edit elasticsearch.service
[Service]
LimitMEMLOCK=infinity
```

## 4.db03集群配置文件

```
cat > /etc/elasticsearch/elasticsearch.yml <<EOF
cluster.name: linux5
node.name: node-3
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
bootstrap.memory_lock: true
network.host: 10.0.0.53,127.0.0.1
http.port: 9200
discovery.zen.ping.unicast.hosts: ["10.0.0.51", "10.0.0.53"]
discovery.zen.minimum_master_nodes: 1
EOF
```

## 5.添加节点注意

```
1.对于新添加的节点来说:  
只需要直到集群内任意一个节点的IP和他自己本身的IP即可

对于以前的节点来说:
什么都不需要更改

2.最大master节点数设置
3个节点,设置为2

3.默认创建索引为1副本5分片

4.数据分配的时候会出现2中颜色
紫色: 正在迁移
黄色: 正在复制
绿色: 正常

5.3节点的时候
0副本一台都不能坏 
1副本的极限情况下可以坏2台: 1台1台的坏,不能同时坏2台,在数据复制完成的情况下,可以坏2台
2副本的情况可以同时坏2台
```

# 第13章 动态修改最小发现节点数

```
GET _cluster/settings

PUT _cluster/settings
{
  "transient": {
    "discovery.zen.minimum_master_nodes": 2
  }
}
```

# 第14章 自定义副本分片和索引

## 1.注意事项

```
索引一旦建立完成,分片数就不可以修改了
但是副本数可以随时修改
```

## 2.创建索引的时候就自定义副本和分片

```
PUT /yayayaay/
{
  "settings": {
    "number_of_shards": 3, 
    "number_of_replicas": 0
  }
}
```

## 3.修改单个索引的副本数

```
PUT /oldzhang/_settings/
{
  "settings": {
    "number_of_replicas": 0
  }
}
```

## 4.修改所有的索引的副本数

```
PUT /_all/_settings/
{
  "settings": {
    "number_of_replicas": 0
  }
}
```

## 5.工作如何设置

```
2个节点: 默认就可以 
3个节点: 重要的数据,2副本 不重要的默认 
日志收集: 1副本3分片 
```

# 第15章 ES监控

## 1.监控注意

```
0.不能只监控集群状态
1.监控节点数
2.监控集群状态
3.两者任意一个发生改变了都报警
```

## 2.监控命令

```
GET _cat/nodes
GET _cat/health
```

# 第16章 ES优化

```
1.内存 
不要超过32G 

48内存 
系统留一半: 24G 
自己留一半: 24G
8G 12G 16G 24G 30G 

2.SSD硬盘
0   1

3.代码优化

4.升级大版本
```

# 第15章 中文分词

## 未分词的情况

### 1.插入测试数据

```
POST /news/txt/1
{"content":"美国留给伊拉克的是个烂摊子吗"}

POST /news/txt/2
{"content":"公安部：各地校车将享最高路权"}

POST /news/txt/3
{"content":"中韩渔警冲突调查：韩警平均每天扣1艘中国渔船"}

POST /news/txt/4
{"content":"中国驻洛杉矶领事馆遭亚裔男子枪击 嫌犯已自首"}
```

### 2.检测

```
POST /news/_search
{
    "query" : { "match" : { "content" : "中国" }},
    "highlight" : {
        "pre_tags" : ["<tag1>", "<tag2>"],
        "post_tags" : ["</tag1>", "</tag2>"],
        "fields" : {
            "content" : {}
        }
    }
}
```

## 中文分词配置

### 0.前提条件

```
所有的ES节点都需要安装
所有的ES都需要重启才能生效
```

### 1.配置中文分词器

在线安装

```
cd /usr/share/elasticsearch
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.6.0/elasticsearch-analysis-ik-6.6.0.zip
```

离线本地文件安装

```
/usr/share/elasticsearch/bin/elasticsearch-plugin install file:///XXX/elasticsearch-analysis-ik-6.6.0.zip
```

### 2.创建索引

```
PUT /news2
```

### 3.创建模板

```
POST /news2/text/_mapping
{
        "properties": {
            "content": {
                "type": "text",
                "analyzer": "ik_max_word",
                "search_analyzer": "ik_smart"
            }
        }

}
```

### 4.插入测试数据

```
POST /news2/text/1
{"content":"美国留给伊拉克的是个烂摊子吗"}

POST /news2/text/2
{"content":"公安部：各地校车将享最高路权"}

POST /news2/text/3
{"content":"中韩渔警冲突调查：韩警平均每天扣1艘中国渔船"}

POST /news2/text/4
{"content":"中国驻洛杉矶领事馆遭亚裔男子枪击 嫌犯已自首"}
```

### 5.再次查询数据发现已经能识别中文了

```
POST /news2/_search
{
    "query" : { "match" : { "content" : "中国" }},
    "highlight" : {
        "pre_tags" : ["<tag1>", "<tag2>"],
        "post_tags" : ["</tag1>", "</tag2>"],
        "fields" : {
            "content" : {}
        }
    }
}
```

## 热更新中文分词库

#### 1.安装nginx

```
cat >> /etc/yum.repos.d/nginx.repo<<EOF
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/\$releasever/\$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true

[nginx-mainline]
name=nginx mainline repo
baseurl=http://nginx.org/packages/mainline/centos/\$releasever/$basearch/
gpgcheck=1
enabled=0
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
EOF

yum makecache fast
yum install nginx -y
```

#### 2.编写字典文件

```
cat >>/usr/share/nginx/html/my_dic.txt<<EOF
上海
班长
学委
张亚
胖虎
EOF
```

#### 3.重启Nginx

```
nginx -t
systemctl restart nginx 
```

#### 4.配置es的中文分词器插件

```
vim /etc/elasticsearch/analysis-ik/IKAnalyzer.cfg.xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
    <comment>IK Analyzer 扩展配置</comment>
    <!--用户可以在这里配置自己的扩展字典 -->
    <entry key="ext_dict"></entry>
     <!--用户可以在这里配置自己的扩展停止词字典-->
    <entry key="ext_stopwords"></entry>
    <!--用户可以在这里配置远程扩展字典 -->
    <entry key="remote_ext_dict">http://10.0.0.51/my_dic.txt</entry>
    <!--用户可以在这里配置远程扩展停止词字典-->
    <!-- <entry key="remote_ext_stopwords">words_location</entry> -->
</properties>
```

#### 5.将修改好的IK配置文件复制到其他所有ES节点

```
cd /etc/elasticsearch/analysis-ik/
scp IKAnalyzer.cfg.xml 10.0.0.52:/etc/elasticsearch/analysis-ik/
```

#### 6.重启所有的ES节点

```
systemctl restart elasticsearch.service 
```

#### 7.查看日志里字典的词有没有加载出来

```
[2020-02-12T14:56:38,610][INFO ][o.w.a.d.Monitor          ] [node-1] 重新加载词典...
[2020-02-12T14:56:38,611][INFO ][o.w.a.d.Monitor          ] [node-1] try load config from /etc/elasticsearch/analysis-ik/IKAnalyzer.cfg.xml
[2020-02-12T14:56:38,614][INFO ][o.w.a.d.Monitor          ] [node-1] [Dict Loading] http://10.0.0.51/my_dic.txt
[2020-02-12T14:56:38,628][INFO ][o.w.a.d.Monitor          ] [node-1] 上海
[2020-02-12T14:56:38,629][INFO ][o.w.a.d.Monitor          ] [node-1] 班长
[2020-02-12T14:56:38,629][INFO ][o.w.a.d.Monitor          ] [node-1] 学委
[2020-02-12T14:56:38,629][INFO ][o.w.a.d.Monitor          ] [node-1] 张亚
[2020-02-12T14:56:38,629][INFO ][o.w.a.d.Monitor          ] [node-1] 胖虎
[2020-02-12T14:56:38,629][INFO ][o.w.a.d.Monitor          ] [node-1] 重新加载词典完毕...
```

#### 8.打开es日志，然后更新字典内容，查看日志里会不会自动加载

```
echo "武汉" >> /usr/share/nginx/html/my_dic.txt
```

#### 9.搜索测试验证结果

```
POST /news2/text/7
{"content":"武汉加油！"}


POST /news2/_search
{
    "query" : { "match" : { "content" : "武汉" }},
    "highlight" : {
        "pre_tags" : ["<tag1>", "<tag2>"],
        "post_tags" : ["</tag1>", "</tag2>"],
        "fields" : {
            "content" : {}
        }
    }
}
```

#### 10.电商上架新产品流程

```
先把新上架的商品的关键词更新到词典里
查看ES日志，确认新词被动态更新了
自己编写一个测试索引，插入测试数据，然后查看搜索结果
确认没有问题之后，在让开发插入新商品的数据
测试
```

# 第17章 备份恢复

## 1.前提条件

需要node环境

```
npm -v
node -v
```

## 2.nodejs安装

```
https://nodejs.org/dist/v10.16.3/node-v10.16.3-linux-x64.tar.xz
tar xf  node-v10.16.3-linux-x64.tar.xz -C /opt/node
echo "export PATH=/opt/node/bin:\$PATH" >> /etc/profile
source /etc/profile
npm -v
node -v
```

## 3.指定使用国内淘宝npm源

```
npm install -g cnpm --registry=https://registry.npm.taobao.org
```

## 4.安装es-dump

```
cnpm install elasticdump -g
```

## 5.备份

备份成可读的json格式

```
elasticdump \
  --input=http://10.0.0.51:9200/news2 \
  --output=/data/news2.json \
  --type=data
```

备份成压缩格式

```
elasticdump \
  --input=http://10.0.0.51:9200/news2 \
  --output=$|gzip > /data/news2.json.gz  
```

备份分词器/mapping/数据一条龙服务

```
elasticdump \
  --input=http://10.0.0.51:9200/news2 \
  --output=/data/news2_analyzer.json \
  --type=analyzer
elasticdump \
  --input=http://10.0.0.51:9200/news2 \
  --output=/data/news2_mapping.json \
  --type=mapping
elasticdump \
  --input=http://10.0.0.51:9200/news2 \
  --output=/data/news2.json \
  --type=data
```

## 6.恢复

只恢复数据

```
elasticdump \
  --input=/data/news2.json \
  --output=http://10.0.0.51:9200/news2
```

恢复所有数据包含分词器/mapping一条龙

```
elasticdump \
  --input=/data/news2_analyzer.json \
  --output=http://10.0.0.51:9200/news2 \
  --type=analyzer
elasticdump \
  --input=/data/news2_mapping.json \
  --output=http://10.0.0.51:9200/news2 \
  --type=mapping
elasticdump \
  --input=/data/news2.json \
  --output=http://10.0.0.51:9200/news2
  --type=data
```

## 7.批量备份

```
curl -s 127.0.0.1:9200/_cat/indices|awk '{print $3}'
```