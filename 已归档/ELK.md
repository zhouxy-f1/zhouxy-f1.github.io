# 第1章 ELK简介

```
E:  elasticsearch   存储数据               java
L:  logstash        收集,过滤,转发,匹配 		java
K:  kibana          过滤,分析,图形展示      java
F:  filebeat        收集日志,过滤           go
```

# 第2章: 传统日志分析需求

```
1.找出访问网站频次最高的 IP 排名前十
2.找出访问网站排名前十的 URL
3.找出中午 10 点到 2 点之间 www 网站访问频次最高的 IP
4.对比昨天这个时间段和今天这个时间段访问频次有什么变化
5.对比上周这个时间和今天这个时间的区别
6.找出特定的页面被访问了多少次
7.找出有问题的 IP 地址，并告诉我这个 IP 地址都访问了什么页面，在对比前几天他来过吗？他从什么时间段开
始访问的，什么时间段走了
8.找出来访问最慢的前十个页面并统计平均响应时间，对比昨天这也页面访问也这么慢吗？
9.找出搜索引擎今天各抓取了多少次？抓取了哪些页面？响应时间如何？
10.找出伪造成搜索引擎的 IP 地址
11.5 分钟之内告诉我结果
```

# 第3章: 日志收集分类

```
代理层: nginx haproxy
web层:  nginx tomcat java php
db层:   mysql mongo redis es 
系统层: message secure
```

# 第4章 准备ES单机环境

```
cat >/etc/elasticsearch/elasticsearch.yml <<EOF
node.name: node-1
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
bootstrap.memory_lock: true
network.host: 10.0.0.51,127.0.0.1
http.port: 9200
EOF

systemctl stop elasticsearch
systemctl stop kibana
rm -rf /var/lib/elasticsearch/*
rm -rf /var/lib/kibana/*
systemctl start elasticsearch
systemctl start kibana
netstat -lntup|grep 9200
netstat -lntup|grep 5601
```

# 第5章 filebeat收集Nginx普通格式日志

## 0.更新系统时间

```
ntpdate time1.aliyun.com
```

## 1.安装nginx

```
[root@db-01 ~]# cat /etc/yum.repos.d/nginx.repo 
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=0
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key

[nginx-mainline]
name=nginx mainline repo
baseurl=http://nginx.org/packages/mainline/centos/$releasever/$basearch/
gpgcheck=0
enabled=0
gpgkey=https://nginx.org/keys/nginx_signing.key

yum makecache fast
yum install nginx -y
systemctl start nginx
```

## 2.安装filebeat

```
rpm -ivh filebeat-6.6.0-x86_64.rpm
rpm -qc filebeat
```

## 3.配置filebeat

```
cp  /etc/filebeat/filebeat.yml /opt/
cat >/etc/filebeat/filebeat.yml<<EOF
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/nginx/access.log
output.elasticsearch:
  hosts: ["10.0.0.51:9200"]
EOF    
```

## 5.启动并检查

```
systemctl start filebeat
tail -f /var/log/filebeat/filebeat
```

## 6.查看日志结果

```
es-head查看
```

## 7.kibana添加

```
Management >> Index Patterns >> filebeat-6.6.0-2019.11.15 >>@timestamp >>create >> discover
```

# 第6章: filebeat收集Nginx的json格式日志

## 1.上面方案不完善的地方

```
所有日志都存储在message的value里,不能拆分单独显示
```

## 2.理想中的情况

```
可以把日志所有字段拆分出来
{
    $remote_addr : 192.168.12.254
    - : -
    $remote_user : -
    [$time_local]: [10/Sep/2019:10:52:08 +0800]
    $request: GET /jhdgsjfgjhshj HTTP/1.0
    $status : 404
    $body_bytes_sent : 153
    $http_referer : -
    $http_user_agent :ApacheBench/2.3
    $http_x_forwarded_for:-
}
```

## 3.目标

```
如何使nginx日志格式转换成我们想要的json格式
```

## 4.修改nginx配置文件使日志转换成json

```
log_format json '{ "time_local": "$time_local", '
                          '"remote_addr": "$remote_addr", '
                          '"referer": "$http_referer", '
                          '"request": "$request", '
                          '"status": $status, '
                          '"bytes": $body_bytes_sent, '
                          '"agent": "$http_user_agent", '
                          '"x_forwarded": "$http_x_forwarded_for", '
                          '"up_addr": "$upstream_addr",'
                          '"up_host": "$upstream_http_host",'
                          '"upstream_time": "$upstream_response_time",'
                          '"request_time": "$request_time"'
    ' }';
    access_log  /var/log/nginx/access.log  json;
```

清除旧日志

```
> /var/log/nginx/access.log
```

检查并重启nginx

```
nginx -t 
systemctl restart nginx
```

## 5.nginx转换成json之后仍然不完善的地方

```
通过查看发现,虽然nginx日志变成了json,但是es里还是存储在message里仍然不能拆分
```

## 6.目标

```
如何在ES里展示的是json格式
```

## 7.修改filebeat配置文件支持json解析

```
cat >/etc/filebeat/filebeat.yml<<EOF
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/nginx/access.log
  json.keys_under_root: true
  json.overwrite_keys: true

output.elasticsearch:
  hosts: ["10.0.0.51:9200"]
EOF
```

## 7.删除ES里以前的索引

```
es-head >> filebeat-6.6.0-2019.11.15 >> 动作 >>删除 
```

## 8.重启filebeat

```
systemctl restart filebeat
```

# 第7章 filebeat自定义ES索引名称

## 1.理想中的索引名称

```
nginx-6.6.0-2019.11.15
```

## 2.filebeat配置

```
cat >/etc/filebeat/filebeat.yml<<EOF
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/nginx/access.log
  json.keys_under_root: true
  json.overwrite_keys: true

output.elasticsearch:
  hosts: ["10.0.0.51:9200"]
  index: "nginx-%{[beat.version]}-%{+yyyy.MM}"
setup.template.name: "nginx"
setup.template.pattern: "nginx-*"
setup.template.enabled: false
setup.template.overwrite: true
EOF 
```

# 第8章 filebeat按照服务类型拆分索引

## 1.第一种写法

```
cat >/etc/filebeat/filebeat.yml<<EOF
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/nginx/access.log
  json.keys_under_root: true
  json.overwrite_keys: true

- type: log
  enabled: true
  paths:
    - /var/log/nginx/error.log

output.elasticsearch:
  hosts: ["10.0.0.51:9200"]
  indices:
    - index: "nginx-access-%{[beat.version]}-%{+yyyy.MM}"
      when.contains:
        source: "/var/log/nginx/access.log"
    - index: "nginx-error-%{[beat.version]}-%{+yyyy.MM}"
      when.contains:
        source: "/var/log/nginx/error.log"

setup.template.name: "nginx"
setup.template.pattern: "nginx-*"
setup.template.enabled: false
setup.template.overwrite: true
EOF 
```

## 2.第二种写法：

```
cat >/etc/filebeat/filebeat.yml<<EOF 
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/nginx/access.log
  json.keys_under_root: true
  json.overwrite_keys: true
  tags: ["access"]

- type: log
  enabled: true
  paths:
    - /var/log/nginx/error.log
  tags: ["error"]

output.elasticsearch:
  hosts: ["10.0.0.51:9200"]
  indices:
    - index: "nginx-access-%{[beat.version]}-%{+yyyy.MM}"
      when.contains:
        tags: "access"
    - index: "nginx-error-%{[beat.version]}-%{+yyyy.MM}"
      when.contains:
        tags: "error"

setup.template.name: "nginx"
setup.template.pattern: "nginx-*"
setup.template.enabled: false
setup.template.overwrite: true
EOF
```

# 第9章 filebeat收集tomcat的json日志

## 1.安装tomcat

## 2.配置tomcat日志格式为json

```
[root@web01 ~]# /opt/tomcat/bin/shutdown.sh
[root@web01 ~]# sed -n '162p' /opt/tomcat/conf/server.xml 
           pattern="{&quot;clientip&quot;:&quot;%h&quot;,&quot;ClientUser&quot;:&quot;%l&quot;,&quot;authenticated&quot;:&quot;%u&quot;,&quot;AccessTime&quot;:&quot;%t&quot;,&quot;method&quot;:&quot;%r&quot;,&quot;status&quot;:&quot;%s&quot;,&quot;SendBytes&quot;:&quot;%b&quot;,&quot;Query?string&quot;:&quot;%q&quot;,&quot;partner&quot;:&quot;%{Referer}i&quot;,&quot;AgentVersion&quot;:&quot;%{User-Agent}i&quot;}"/>
```

## 3.启动tomcat

```
/opt/tomcat/bin/startup.sh
```

## 4.配置filebeat

```
cat >/etc/filebeat/filebeat.yml <<EOF
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /opt/tomcat/logs/localhost_access_log.*.txt
  json.keys_under_root: true
  json.overwrite_keys: true
  tags: ["tomcat"]

output.elasticsearch:
  hosts: ["10.0.0.51:9200"]
  index: "tomcat_access-%{[beat.version]}-%{+yyyy.MM}"

setup.template.name: "tomcat"
setup.template.pattern: "tomcat_*"
setup.template.enabled: false
setup.template.overwrite: true
EOF
```

## 5.重启filebeat

```
systemctl restart filebeat
```

## 6.访问tomcat查看是否有数据生成

# 第10章 filebeat收集java多行匹配模式

## 1.filebeat配置文件

```
[root@db01 ~]# cat /etc/filebeat/filebeat.yml 
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/elasticsearch/elasticsearch.log
  multiline.pattern: '^\['
  multiline.negate: true
  multiline.match: after

output.elasticsearch:
  hosts: ["10.0.0.51:9200"]
  index: "es-%{[beat.version]}-%{+yyyy.MM}"
setup.template.name: "es"
setup.template.pattern: "es-*"
setup.template.enabled: false
setup.template.overwrite: true
```

# 第11章 filbeat使用模块收集nginx日志

## 1.删除以前的es索引和kibana索引

## 2.确认Nginx日式知否为普通格式

```
systemctl stop nginx 
rm -rf /var/log/nginx/* 
自己修改日志格式为main的普通格式
systemctl start nginx
```

## 3.安装nginx模块所需的插件

```
cd /usr/share/elasticsearch/
./bin/elasticsearch-plugin install file:///root/ingest-geoip-6.6.0.zip 
./bin/elasticsearch-plugin install file:///root/ingest-user-agent-6.6.0.zip
systemctl restart elasticsearch
```

## 4.检查filebeat配置文件里是否包含模块相关参数

```
filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: true 
  reload.period: 10s
```

## 5.激活filebeat模块并查看

```
filebeat modules --list 
filebeat enable nginx 
```

## 6.配置filebeat的nginx模块

```
[root@web01 ~]# cat /etc/filebeat/modules.d/nginx.yml 
- module: nginx
  access:
    enabled: true
    var.paths: ["/var/log/nginx/*.log"]

  error:
    enabled: true
    var.paths: ["/var/log/nginx/error.log"]
```

## 7.filebeat配置

```
cat >/etc/filebeat/filebeat.yml<<EOF
filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: true 
  reload.period: 10s

output.elasticsearch:
  hosts: ["10.0.0.51:9200"]
  indices:
  - index: "nginx-www-%{[beat.version]}-%{+yyyy.MM}"
    when.contains:
      source: "/var/log/nginx/www.log"
  - index: "nginx-blog-%{[beat.version]}-%{+yyyy.MM}"
    when.contains:
      source: "/var/log/nginx/blog.log"
  - index: "nginx-error-%{[beat.version]}-%{+yyyy.MM}"
    when.contains:
      source: "/var/log/nginx/error.log"

setup.template.name: "nginx"
setup.template.pattern: "nginx-*"
setup.template.enabled: false
setup.template.overwrite: true
EOF
```

## 8.重启filebeat

```
systemctl restart filebeat
```

## 9.访问nginx生成测试日志

# 第12章 filebeat使用模块收集mysql慢日志

## 1.配置mysql错误日志和慢日志路径

```
编辑my.cnf
[mysqld]
slow_query_log=ON
slow_query_log_file=/var/log/mariadb/slow.log
long_query_time=1
```

## 2.重启mysql并制造慢日志

```
systemctl restart mysql 
慢日志制造语句
select sleep(2) user,host from mysql.user ;
```

## 3.确认慢日志和错误日志确实有生成

```
mysql -uroot -poldboy123 -e "show variables like '%slow_query_log%'"
```

## 4.激活filebeat的mysql模块

```
filebeat module enable mysql
```

## 5.配置mysql的模块

```
module: mysql
error:
  enabled: true
  var.paths: ["/var/log/mariadb/mariadb.log"]

slowlog:
  enabled: true 
  var.paths: ["/var/log/mariadb/slow.log"]
```

## 6.配置filebeat根据日志类型做判断

```
filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: true
  reload.period: 10s

output.elasticsearch:
  hosts: ["10.0.0.51:9200"]
  indices:
    - index: "mysql_slowlog-%{[beat.version]}-%{+yyyy.MM}"
      when.contains:
        fileset.module: "mysql"
        fileset.name: "slowlog"
    - index: "mysql_error-%{[beat.version]}-%{+yyyy.MM}"
      when.contains:
        fileset.module: "mysql"
        fileset.name: "error"

setup.template.name: "mysql"
setup.template.pattern: "mysql_*"
setup.template.enabled: false
setup.template.overwrite: true
```

## 7.重启filebeat

```
systemctl restart filebeat
```

# 第13章 filebeat收集docker日志

## 0.docker安装命令

```
wget -O /etc/yum.repos.d/docker-ce.repo https://mirrors.ustc.edu.cn/docker-ce/linux/centos/docker-ce.repo
sed -i 's#download.docker.com#mirrors.tuna.tsinghua.edu.cn/docker-ce#g' /etc/yum.repos.d/docker-ce.repo
yum install docker-ce -y
systemctl start docker
```

## 1.启动2个nginx容器

```
systemctl stop nginx
pkill java

docker run -d -p 80:80 nginx
docker run -d -p 8080:80 nginx
```

## 2.查看容器日志

```
docker logs -f ce22c2583da5
```

## 3.修改filebeat配置文件

```
cat >>/etc/filebeat/filebeat.yml<<EOF 
filebeat.inputs:
- type: docker
  containers.ids: 
    - '*'

output.elasticsearch:
  hosts: ["10.0.0.51:9200"]
  index: "docker-%{[beat.version]}-%{+yyyy.MM}"

setup.template.name: "docker"
setup.template.pattern: "docker-*"
setup.template.enabled: false
setup.template.overwrite: true
EOF
```

## 4.重启filebeat

```
systemctl restart filebeat
```

## 5.访问nginx制造日志

```
curl 127.0.0.1/11111111111111111111
curl 127.0.0.1:8080/22222222222222222222
```

# 第14章 filebeat收集docker日志可以早下班版

## 1.理想中的索引:

```
docker-mysql-xxxx
docker-nginx-xxxx
```

## 2.理想中的日志格式:

```
{
    "log": "10.0.0.1 - - [18/Nov/2019:02:16:44 +0000] \"GET /web01 HTTP/1.1\" 404 555 \"-\" \"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/74.0.3729.131 Safari/537.36\" \"-\"\n",
    "stream": "stdout",
    "time": "2019-11-18T02:16:44.010910131Z",
    "service": "nginx"
}
```

## 3..安装docker-compose

```
yum install docker-compose -y
```

## 4.编写docker-compose文件

```
cat >docker-compose.yml<<EOF
version: '3'
services:
  nginx:
    image: nginx:latest
    labels:
      service: nginx
    logging:
      options:
        labels: "service"
    ports:
      - "80:80"
  db:
    image: nginx:latest
    labels:
      service: db 
    logging:
      options:
        labels: "service"
    ports:
      - "8080:80"
EOF
```

## 5.删除旧的容器

```
docker stop $(docker ps -q)
docker rm $(docker ps -qa)
```

## 6.启动docker-compose

```
docker-compose up -d
```

## 7.修改filebeat配置文件

```
cat >/etc/filebeat/filebeat.yml<<EOF 
filebeat.inputs:
- type: log 
  enabled: true
  paths:
    - /var/lib/docker/containers/*/*-json.log
  json.keys_under_root: true
  json.overwrite_keys: true

output.elasticsearch:
  hosts: ["10.0.0.51:9200"]
  indices:
    - index: "docker-nginx-%{[beat.version]}-%{+yyyy.MM}"
      when.contains:
        attrs.service: "nginx"
    - index: "docker-db-%{[beat.version]}-%{+yyyy.MM}"
      when.contains:
        attrs.service: "db"

setup.template.name: "docker"
setup.template.pattern: "docker-*"
setup.template.enabled: false
setup.template.overwrite
EOF
```

## 8.重启filebeat

```
systemctl restart filebeat
```

## 9.生成测试命令

```
curl 127.0.0.1/nginxxxxxxxx
curl 127.0.0.1:8080/dbbbbbbbbbbbbb
```

# 第15章 filebeat收集docker日志升职加薪版

## 1.分析正常日志和错误日志字段的区别

```
错误日志字段： stream：stderr
正常日志字段： stream：stdout
```

## 2.修改filebeat配置文件

```
cat >/etc/filebeat/filebeat.yml<EOF
filebeat.inputs:
- type: log 
  enabled: true
  paths:
    - /var/lib/docker/containers/*/*-json.log
  json.keys_under_root: true
  json.overwrite_keys: true

output.elasticsearch:
  hosts: ["10.0.0.51:9200"]
  indices:
    - index: "docker-nginx-access-%{[beat.version]}-%{+yyyy.MM}"
      when.contains:
        stream: "stdout"
        attrs.service: "nginx"
    - index: "docker-nginx-error-%{[beat.version]}-%{+yyyy.MM}"
      when.contains:
        stream: "stderr"
        attrs.service: "nginx"
    - index: "docker-db-access-%{[beat.version]}-%{+yyyy.MM}"
      when.contains:
        stream: "stdout"
        attrs.service: "db"
    - index: "docker-db-error-%{[beat.version]}-%{+yyyy.MM}"
      when.contains:
        stream: "stderr"
        attrs.service: "db"

setup.template.name: "docker"
setup.template.pattern: "docker-*"
setup.template.enabled: false
setup.template.overwrite: true
EOF
```

## 3.重启filebeat

```
systemctl restart filebeat
```

## 4.生成测试命令

```
curl 127.0.0.1/nginxxxxxxxx
curl 127.0.0.1:8080/dbbbbbbbbbbbbb
```

# 第16章: filebeat收集docker日志终极杀人王版

## 0.创建容器日志目录

```
mkdir /opt/{nginx,mysql}
```

## 1.将容器的日志目录挂载到宿主机

```
docker ps 
docker cp 容器ID:/etc/nginx/nginx.conf .
修改nginx配置文件里的日志记录类型为json格式 
docker cp /etc/nginx/nginx.conf 容器ID:/etc/nginx/nginx.conf
docker commit 容器ID nginx:v2
docker-compose stop
docker rm -f  $(docker ps -a -q)
docker run -d -p 80:80 -v /opt/nginx:/var/log/nginx nginx:v2
docker run -d -p 8080:80 -v /opt/mysql:/var/log/nginx nginx:v2
```

## 2.修改filebeat配置文件

```
cat >/etc/filebeat/filebeat.yml<<EOF
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /opt/nginx/access.log
  json.keys_under_root: true
  json.overwrite_keys: true
  tags: ["nginx_access"]

- type: log
  enabled: true
  paths:
    - /opt/nginx/error.log
  tags: ["nginx_error"]

- type: log
  enabled: true
  paths:
    - /opt/mysql/access.log
  json.keys_under_root: true
  json.overwrite_keys: true
  tags: ["mysql_access"]

- type: log
  enabled: true
  paths:
    - /opt/mysql/error.log
  tags: ["mysql_error"]

output.elasticsearch:
  hosts: ["10.0.0.51:9200"]
  indices:
    - index: "nginx-access-%{[beat.version]}-%{+yyyy.MM}"
      when.contains:
        tags: "nginx_access"
    - index: "nginx-error-%{[beat.version]}-%{+yyyy.MM}"
      when.contains:
        tags: "nginx_error"
    - index: "mysql-access-%{[beat.version]}-%{+yyyy.MM}"
      when.contains:
        tags: "mysql_access"
    - index: "mysql-error-%{[beat.version]}-%{+yyyy.MM}"
      when.contains:
        tags: "mysql_error"

setup.template.name: "nginx"
setup.template.pattern: "nginx-*"
setup.template.enabled: false
setup.template.overwrite: true
EOF
```

## 3.重启filebeat

```
systemctl restart filebeat
```

## 4.生成测试命令

```
curl 127.0.0.1/nginxxxxxxxx
curl 127.0.0.1:8080/dbbbbbbbbbbbbb
```

# 第17章 使用redis缓存服务来缓解ES压力

## 1.安装redis

```
yum install redis 
sed -i 's#^bind 127.0.0.1#bind 127.0.0.1 10.0.0.51#' /etc/redis.conf
systemctl start redis 
netstat -lntup|grep redis 
redis-cli -h 10.0.0.51 
```

## 2.停止docker

```
systemctl stop docker.service
```

## 3.配置filebeat

```
cat >/etc/filebeat/filebeat.yml<<EOF 
filebeat.inputs:
- type: log
  enabled: true 
  paths:
    - /var/log/nginx/www.log
  json.keys_under_root: true
  json.overwrite_keys: true
  tags: ["access"]

- type: log
  enabled: true 
  paths:
    - /var/log/nginx/error.log
  tags: ["error"]

output.redis:
  hosts: ["10.0.0.51"]
  keys:
    - key: "nginx_access"
      when.contains:
        tags: "access"
    - key: "nginx_error"
      when.contains:
        tags: "error"

setup.template.name: "nginx"
setup.template.pattern: "nginx_*"
setup.template.enabled: false
setup.template.overwrite: true
EOF
```

## 4.设置nginx日志为json格式

```
systemctl stop nginx 
> /var/log/nginx/www.log 
cat >/etc/nginx/conf.d/www.conf <<EOF
server {
    listen 80;
    server_name www.mysun.com; 
    access_log  /var/log/nginx/www.log  json;
    location / {
        root   /code/www;
        index  index.html index.htm;
    }
}
EOF
mkdir -p /code/www/
echo "web01 www" > /code/www/index.html
nginx -t
systemctl start nginx 
echo "10.0.0.51 www.mysun.com" >> /etc/hosts 
curl www.mysun.com/www 
tail -f /var/log/nginx/www.log 
```

## 5.重启filebeat

```
systemctl restart filebeat 
```

## 6.检查redis是否有数据

```
redis-cli LRANGE nginx_access 0 -1 
```

## 7.配置logstash

```
yum install java -y 
cat >/etc/logstash/conf.d/redis.conf <<EOF
input {
  redis {
    host => "10.0.0.51"
    port => "6379"
    db => "0"
    key => "nginx_access"
    data_type => "list"
  }
  redis {
    host => "10.0.0.51"
    port => "6379"
    db => "0"
    key => "nginx_error"
    data_type => "list"
  }
}

filter {
  mutate {
    convert => ["upstream_time", "float"]
    convert => ["request_time", "float"]
  }
}

output {
   stdout {}
   if "access" in [tags] {
      elasticsearch {
        hosts => "http://10.0.0.51:9200"
        manage_template => false
        index => "nginx_access-%{+yyyy.MM}"
      }
    }
    if "error" in [tags] {
      elasticsearch {
        hosts => "http://10.0.0.51:9200"
        manage_template => false
        index => "nginx_error-%{+yyyy.MM}"
      }
    }
}
EOF
```

## 8.前台启动Logstash测试

```
删除ES旧的索引
/usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/redis.conf
```

## 9.生成测试数据

```
yum install httpd-tools -y 
ab -c 100 -n 2000 www.mysun.com/www 
redis-cli LLEN nginx_access
```

## 10.如果数据正常传输给了es,在后台启动logstash

```
systemctl start logstash 
```

# 第18章 使用kafka作为缓存

## 1.配置hosts和密钥

```
cat >/etc/hosts<<EOF
10.0.0.51 db01
10.0.0.52 db02
10.0.0.53 db03
EOF
ssh-keygen
ssh-copy-id 10.0.0.52
ssh-copy-id 10.0.0.53
```

## 2.安装配置zookeeper

db01操作

```
cd /data/soft
tar zxf zookeeper-3.4.11.tar.gz -C /opt/
ln -s /opt/zookeeper-3.4.11/ /opt/zookeeper
mkdir -p /data/zookeeper
cp /opt/zookeeper/conf/zoo_sample.cfg /opt/zookeeper/conf/zoo.cfg
cat >/opt/zookeeper/conf/zoo.cfg<<EOF
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/data/zookeeper
clientPort=2181
server.1=10.0.0.51:2888:3888
server.2=10.0.0.52:2888:3888
server.3=10.0.0.53:2888:3888
EOF
echo "1" > /data/zookeeper/myid
cat /data/zookeeper/myid
rsync -avz /opt/zookeeper* 10.0.0.52:/opt/
rsync -avz /opt/zookeeper* 10.0.0.53:/opt/
```

db02操作

```
mkdir -p /data/zookeeper
echo "2" > /data/zookeeper/myid
cat /data/zookeeper/myid
```

db03操作

```
mkdir -p /data/zookeeper
echo "3" > /data/zookeeper/myid
cat /data/zookeeper/myid
```

## 3.所有节点启动zookeeper

```
/opt/zookeeper/bin/zkServer.sh start
```

## 4.每个节点都检查

```
/opt/zookeeper/bin/zkServer.sh status
```

## 5.测试zookeeper

在一个节点上执行,创建一个频道

```
/opt/zookeeper/bin/zkCli.sh -server 10.0.0.51:2181
create /test "hello"
```

在其他节点上看能否接收到

```
/opt/zookeeper/bin/zkCli.sh -server 10.0.0.52:2181
get /test
```

## 6.安装部署kafka

### db01操作

```
cd /data/soft/
tar zxf kafka_2.11-1.0.0.tgz -C /opt/
ln -s /opt/kafka_2.11-1.0.0/ /opt/kafka
mkdir /opt/kafka/logs
cat >/opt/kafka/config/server.properties<<EOF
broker.id=1
listeners=PLAINTEXT://10.0.0.51:9092
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
log.dirs=/opt/kafka/logs
num.partitions=1
num.recovery.threads.per.data.dir=1
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1
log.retention.hours=24
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
zookeeper.connect=10.0.0.51:2181,10.0.0.52:2181,10.0.0.53:2181
zookeeper.connection.timeout.ms=6000
group.initial.rebalance.delay.ms=0
EOF
rsync -avz /opt/kafka* 10.0.0.52:/opt/
rsync -avz /opt/kafka* 10.0.0.53:/opt/
```

### db02操作

```
sed -i "s#10.0.0.51:9092#10.0.0.52:9092#g" /opt/kafka/config/server.properties
sed -i "s#broker.id=1#broker.id=2#g" /opt/kafka/config/server.properties
```

### db03操作

```
sed -i "s#10.0.0.51:9092#10.0.0.53:9092#g" /opt/kafka/config/server.properties
sed -i "s#broker.id=1#broker.id=3#g" /opt/kafka/config/server.properties
```

## 7.前台启动测试

```
/opt/kafka/bin/kafka-server-start.sh  /opt/kafka/config/server.properties
```

## 8.验证进程

```
jps
```

## 9.测试创建topic

```
/opt/kafka/bin/kafka-topics.sh --create  --zookeeper 10.0.0.51:2181,10.0.0.52:2181,10.0.0.53:2181 --partitions 3 --replication-factor 3 --topic kafkatest
```

## 10.测试获取toppid

```
/opt/kafka/bin/kafka-topics.sh --describe --zookeeper 10.0.0.51:2181,10.0.0.52:2181,10.0.0.53:2181 --topic kafkatest
```

## 11.测试删除topic

```
/opt/kafka/bin/kafka-topics.sh --delete --zookeeper 10.0.0.51:2181,10.0.0.52:2181,10.0.0.53:2181 --topic kafkatest
```

## 12.kafka测试命令发送消息

### 1.创建命令

```
/opt/kafka/bin/kafka-topics.sh --create --zookeeper 10.0.0.51:2181,10.0.0.52:2181,10.0.0.53:2181 --partitions 3 --replication-factor 3 --topic  messagetest
```

### 2.测试发送消息

```
/opt/kafka/bin/kafka-console-producer.sh --broker-list  10.0.0.51:9092,10.0.0.52:9092,10.0.0.53:9092 --topic  messagetest
```

### 3.其他节点测试接收

```
/opt/kafka/bin/kafka-console-consumer.sh --zookeeper 10.0.0.51:2181,10.0.0.52:2181,10.0.0.53:2181 --topic messagetest --from-beginning
```

### 4.测试获取所有的频道

```
/opt/kafka/bin/kafka-topics.sh  --list --zookeeper 10.0.0.51:2181,10.0.0.52:2181,10.0.0.53:2181
```

## 13.测试成功之后,可以放在后台启动

```
/opt/kafka/bin/kafka-server-start.sh  -daemon /opt/kafka/config/server.properties
```

## 14.修改filebeat配置文件

```
cat >/etc/filebeat/filebeat.yml <<EOF
filebeat.inputs:
- type: log
  enabled: true 
  paths:
    - /var/log/nginx/access.log
  json.keys_under_root: true
  json.overwrite_keys: true
  tags: ["access"]

- type: log
  enabled: true 
  paths:
    - /var/log/nginx/error.log
  tags: ["error"]

output.kafka:
  hosts: ["10.0.0.51:9092", "10.0.0.52:9092", "10.0.0.53:9092"]
  topic: 'filebeat'

setup.template.name: "nginx"
setup.template.pattern: "nginx_*"
setup.template.enabled: false
setup.template.overwrite: true
EOF
```

## 15.修改logstash配置文件

```
cat >/etc/logstash/conf.d/kafka.conf <<EOF
input {
  kafka{
    bootstrap_servers=>["10.0.0.51:9092,10.0.0.52:9092,10.0.0.53:9092"]
    topics=>["filebeat"]
    #group_id=>"logstash"
    codec => "json"
  }
}

filter {
  mutate {
    convert => ["upstream_time", "float"]
    convert => ["request_time", "float"]
  }
}

output {
   stdout {}
   if "access" in [tags] {
      elasticsearch {
        hosts => "http://10.0.0.51:9200"
        manage_template => false
        index => "nginx_access-%{+yyyy.MM}"
      }
    }
    if "error" in [tags] {
      elasticsearch {
        hosts => "http://10.0.0.51:9200"
        manage_template => false
        index => "nginx_error-%{+yyyy.MM}"
      }
    }
}
EOF
```

## 16.启动logstash并测试

### 1.前台启动

```
/usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/kafka.conf
```

### 2.后台启动

```
systemctl start logstash
```

## 17.logstash移除不需要的字段

在filter区块里添加remove_field字段即可

```
filter {
  mutate {
    convert => ["upstream_time", "float"]
    convert => ["request_time", "float"]
    remove_field => [ "beat" ]
  }
}
```

# 第19章 如何在公司推广ELK

- 优先表达对别人的好处,可以让别人早下班
- 实验环境准备充足,可以随时打开演示,数据和画图丰富一些
- 开发组,后端组,前端组,运维组,DBA组 单独定制面板
- 单独找组长,说优先给咱们组解决问题
- 你看,你有问题还得这么麻烦跑过来,我给你调好之后,你直接点点鼠标就可以了,如果还有问题,您一句话,我过去